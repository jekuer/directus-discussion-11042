<script setup lang="ts">
import api from '@/api';
import { useCollectionsStore } from '@/stores/collections';
import { Collection } from '@/types/collections';
import { translate } from '@/utils/translate-object-values';
import { unexpectedError } from '@/utils/unexpected-error';
import SearchInput from '@/views/private/components/search-input.vue';
import { merge } from 'lodash';
import { computed, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import Draggable from 'vuedraggable';
import SettingsNavigation from '../../../components/navigation.vue';
import CollectionDialog from './components/collection-dialog.vue';
import CollectionItem from './components/collection-item.vue';
import CollectionOptions from './components/collection-options.vue';
import { useExpandCollapse } from './composables/use-expand-collapse';

const { t } = useI18n();

const search = ref<string | null>(null);
const collectionDialogActive = ref(false);
const editCollection = ref<Collection | null>();

const collectionsStore = useCollectionsStore();
const { collapsedIds, hasExpandableCollections, expandAll, collapseAll, toggleCollapse } = useExpandCollapse();

const collections = computed(() => {
	return translate(collectionsStore.allCollections.filter((collection) => collection.meta)).map((collection) => ({
		...collection,
		isCollapsed: collapsedIds.value?.includes(collection.collection),
	}));
});

const rootCollections = computed(() => {
	return collections.value.filter((collection) => !collection.meta?.group);
});

export type CollectionTree = {
	collection: string;
	visible: boolean;
	children: CollectionTree[];
	search: string | null;
	findChild(collection: string): CollectionTree | undefined;
};

function findVisibilityChild(
	collection: string,
	tree: CollectionTree[] = visibilityTree.value,
): CollectionTree | undefined {
	return tree.find((child) => child.collection === collection);
}

const visibilityTree = computed(() => {
	const tree: CollectionTree[] = makeTree();
	const propagateBackwards: CollectionTree[] = [];

	function makeTree(parent: string | null = null): CollectionTree[] {
		const children = collectionsStore.sortedCollections.filter(
			(collection) => (collection.meta?.group ?? null) === parent,
		);

		const normalizedSearch = search.value?.toLowerCase();

		return children.map((collection) => ({
			collection: collection.collection,
			visible: normalizedSearch ? collection.collection.toLowerCase().includes(normalizedSearch) : true,
			search: search.value,
			children: makeTree(collection.collection),
			findChild(collection: string) {
				return findVisibilityChild(collection, this.children);
			},
		}));
	}

	breadthSearch(tree);

	function breadthSearch(tree: CollectionTree[]) {
		for (const collection of tree) {
			if (collection.children.length === 0) continue;

			propagateBackwards.unshift(collection);
		}

		for (const collection of tree) {
			breadthSearch(collection.children);
		}
	}

	for (const child of propagateBackwards) {
		child.visible = child.visible || child.children.some((child) => child.visible);
	}

	return tree;
});

const tableCollections = computed(() =>
	translate(collectionsStore.databaseCollections.filter((collection) => !collection.meta)),
);

const systemCollections = computed(() =>
	translate(collectionsStore.systemCollections.map((collection) => ({ ...collection, icon: 'settings' }))),
);

async function onSort(updates: Collection[], removeGroup = false) {
	const updatesWithSortValue = updates.map((collection, index) =>
		merge(collection, { meta: { sort: index + 1, group: removeGroup ? null : collection.meta?.group } }),
	);

	collectionsStore.collections = collectionsStore.collections.map((collection) => {
		const updatedValues = updatesWithSortValue.find(
			(updatedCollection) => updatedCollection.collection === collection.collection,
		);

		return updatedValues ? merge({}, collection, updatedValues) : collection;
	});

	try {
		api.patch(
			`/collections`,
			updatesWithSortValue.map((collection) => {
				return {
					collection: collection.collection,
					meta: { sort: collection.meta.sort, group: collection.meta.group },
				};
			}),
		);
	} catch (error) {
		unexpectedError(error);
	}
}
</script>

<template>
	<private-view :title="t('settings_data_model')">
		<template #headline>
			<v-breadcrumb :items="[{ name: t('settings'), to: '/settings' }]" />
		</template>

		<template #title-outer:prepend>
			<v-button class="header-icon" rounded icon exact disabled>
				<v-icon name="database" />
			</v-button>
		</template>

		<template #actions>
			<search-input
				v-model="search"
				:show-filter="false"
				:autofocus="collectionsStore.collections.length - systemCollections.length > 25"
				:placeholder="t('search_collection')"
			/>

			<collection-dialog v-model="collectionDialogActive">
				<template #activator="{ on }">
					<v-button v-tooltip.bottom="t('create_folder')" rounded icon secondary @click="on">
						<v-icon name="create_new_folder" />
					</v-button>
				</template>
			</collection-dialog>

			<v-button v-tooltip.bottom="t('create_collection')" rounded icon to="/settings/data-model/+">
				<v-icon name="add" />
			</v-button>
		</template>

		<template #navigation>
			<settings-navigation />
		</template>

		<div class="padding-box">
			<v-info v-if="collections.length === 0" icon="box" :title="t('no_collections')">
				{{ t('no_collections_copy_admin') }}

				<template #append>
					<v-button to="/settings/data-model/+">{{ t('create_collection') }}</v-button>
				</template>
			</v-info>

			<template v-else>
				<transition-expand>
					<div v-if="hasExpandableCollections" class="expand-collapse-button">
						{{ t('expand') }}
						<button @click="expandAll">{{ t('all') }}</button>
						/
						<button @click="collapseAll">{{ t('none') }}</button>
					</div>
				</transition-expand>
				<draggable
					tag="v-list"
					:model-value="rootCollections"
					:group="{ name: 'collections' }"
					:swap-threshold="0.3"
					class="root-drag-container draggable-list"
					item-key="collection"
					handle=".drag-handle"
					v-bind="{ 'force-fallback': true }"
					@update:model-value="onSort($event, true)"
				>
					<template #item="{ element }">
						<collection-item
							:collection="element"
							:collections="collections"
							:is-collapsed="element.isCollapsed"
							:visibility-tree="findVisibilityChild(element.collection)!"
							@edit-collection="editCollection = $event"
							@set-nested-sort="onSort"
							@toggle-collapse="toggleCollapse"
						/>
					</template>
				</draggable>
			</template>

			<v-list class="db-only">
				<v-list-item
					v-for="collection of tableCollections"
					v-show="findVisibilityChild(collection.collection)!.visible"
					:key="collection.collection"
					v-tooltip="t('db_only_click_to_configure')"
					class="collection-row hidden"
					block
					dense
					clickable
				>
					<v-list-item-icon>
						<v-icon name="add" />
					</v-list-item-icon>

					<router-link class="collection-name" :to="`/settings/data-model/${collection.collection}`">
						<v-icon class="collection-icon" name="dns" />
						<span class="collection-name">{{ collection.name }}</span>
					</router-link>

					<collection-options :collection="collection" :has-nested-collections="false" />
				</v-list-item>
			</v-list>

			<v-detail
				v-show="systemCollections.some((collection) => findVisibilityChild(collection.collection)?.visible)"
				:label="t('system_collections')"
			>
				<collection-item
					v-for="collection of systemCollections"
					:key="collection.collection"
					:collection="collection"
					:collections="systemCollections"
					:visibility-tree="findVisibilityChild(collection.collection)!"
					:is-collapsed="false"
					disable-drag
				/>
			</v-detail>
		</div>

		<router-view name="add" />

		<template #sidebar>
			<sidebar-detail icon="info" :title="t('information')" close>
				<div v-md="t('page_help_settings_datamodel_collections')" class="page-description" />
			</sidebar-detail>
		</template>

		<collection-dialog
			:model-value="!!editCollection"
			:collection="editCollection"
			@update:model-value="editCollection = null"
		/>
	</private-view>
</template>

<style scoped lang="scss">
.padding-box {
	padding: var(--content-padding);
	padding-block-start: 0;
}

.v-info {
	padding: var(--content-padding) 0;
}

.root-drag-container {
	padding: 8px 0;
	overflow: hidden;
}

.header-icon {
	--v-button-background-color-disabled: var(--theme--primary-background);
	--v-button-color-disabled: var(--theme--primary);
	--v-button-background-color-hover-disabled: var(--theme--primary-subdued);
	--v-button-color-hover-disabled: var(--theme--primary);
}

.collection-item.hidden {
	--v-list-item-color: var(--theme--foreground-subdued);
}

.collection-icon {
	margin-inline-end: 8px;
}

.hidden .collection-name {
	color: var(--theme--foreground-subdued);
	flex-grow: 1;
}

.draggable-list :deep(.sortable-ghost) {
	.v-list-item {
		--v-list-item-background-color: var(--theme--primary-background);
		--v-list-item-border-color: var(--theme--primary);
		--v-list-item-background-color-hover: var(--theme--primary-background);
		--v-list-item-border-color-hover: var(--theme--primary);

		> * {
			opacity: 0;
		}
	}
}

.db-only {
	margin-block-end: 16px;
}

.expand-collapse-button {
	padding-block: 4px 8px;
	text-align: end;
	color: var(--theme--foreground-subdued);

	button {
		color: var(--theme--foreground-subdued);
		transition: color var(--fast) var(--transition);
	}

	button:hover {
		color: var(--theme--foreground);
		transition: none;
	}
}

.v-list.draggable-list {
	padding-block-start: 0;
}
</style>
