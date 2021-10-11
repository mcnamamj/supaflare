<template>
	<admin-view>
		<h1>Manage Links</h1>
		<n-data-table
			ref="tableRef"
			class="centered-view"
			:row-key="rowKey"
			:loading="loadingRef"
			:columns="columns"
			:data="links"
			:pagination="pagination"
		/>

		<n-modal
			v-model:show="showEditModal"
			preset="card"
			:style="{ width: '600px' }"
			title="Modal"
			:bordered="false"
			size="huge"
			:segmented="{
				content: 'soft',
				footer: 'soft',
			}"
		>
			<template #header>
				<div>Edit Link</div>
			</template>
			<div>
				<n-form ref="formRef" class="centered-form" :model="model" :rules="rules">
					<n-form-item path="url" label="URL">
						<n-input v-model:value="model.url" placeholder="Enter URL" />
					</n-form-item>
					<n-row>
						<n-form-item path="slug" label="Slug" style="flex-grow: 1">
							<n-input v-model:value="model.slug" placeholder="Enter Slug" />
						</n-form-item>
						<n-form-item>
							<n-button type="warning" style="margin-left: 20px" @click="handleGenerateSlug">
								<template #icon>
									<n-icon>
										<sync />
									</n-icon>
								</template>
								Generate Slug
							</n-button>
						</n-form-item>
					</n-row>
					<n-form-item path="android_url" label="Android URL" style="flex-grow: 1">
						<n-input v-model:value="model.android_url" placeholder="Enter Android URL" />
					</n-form-item>
					<n-form-item path="ios_url" label="iOS URL" style="flex-grow: 1">
						<n-input v-model:value="model.ios_url" placeholder="Enter iOS URL" />
					</n-form-item>
				</n-form>
			</div>
			<template #footer>
				<div>
					<n-button type="primary" @click="handleSaveEdits">Update</n-button
					><n-button style="margin-left: 20px">Cancel</n-button>
				</div>
			</template>
		</n-modal>
	</admin-view>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, onMounted, h } from 'vue';
import { fetchLinks, editLink, deleteLink } from '@/services/links';
import { useMessage, useDialog, NDataTable, NButton, NModal, NForm, NFormItem, NInput, NIcon, NRow } from 'naive-ui';
import { Plus, Sync } from '@vicons/fa';
import { useLinksStore } from '@/stores/linksStore';
import { Link } from '@/types/global';
import { customAlphabet } from 'nanoid';

export default defineComponent({
	components: { NDataTable, NModal, NButton, NForm, NFormItem, NInput, NIcon, NRow, Plus, Sync },
	setup() {
		const messageDuration = 5000;
		const linksStore = useLinksStore();
		const message = useMessage();
		const dialog = useDialog();
		const tableRef = ref();
		const loadingRef = ref(true);
		const links = ref<Link[] | []>([]);
		const showEditModal = ref(false);

		const modelRef = ref({
			id: '',
			url: '',
			slug: '',
			android_url: '',
			ios_url: '',
		});
		const editRowRef = ref();

		const rules = {
			url: [
				{
					required: true,
					validator(rule: any, value: any) {
						if (!value) {
							return new Error('URL is required');
						} else if (value.length > 2083) {
							return new Error('URL has to be 2083 characters or below.');
						}
						return true;
					},
					trigger: ['input', 'blur'],
				},
			],
			slug: [
				{
					required: true,
					validator(rule: any, value: any) {
						if (!value) {
							return new Error('Slug is required');
						}
						return true;
					},
					trigger: ['input', 'blur'],
				},
			],
			android_url: [
				{
					validator(rule: any, value: any) {
						if (value && value.length > 2083) {
							return new Error('Android URL has to be 2083 characters or below.');
						}
						return true;
					},
					trigger: ['input', 'blur'],
				},
			],
			ios_url: [
				{
					validator(rule: any, value: any) {
						if (value && value.length > 2083) {
							return new Error('iOS URL has to be 2083 characters or below.');
						}
						return true;
					},
					trigger: ['input', 'blur'],
				},
			],
		};

		// Remove confusion with caps I caps O and l
		const nanoid = customAlphabet('1234567890abcdefghijkmnopqrstuvwxyzABCDEFGHJKLMNPQRSTUVWXYZ', 6);

		async function handleGenerateSlug() {
			modelRef.value.slug = nanoid();
		}

		async function handleSaveEdits() {
			try {
				const { error } = await editLink(editRowRef.value, {
					url: modelRef.value.url,
					slug: modelRef.value.slug,
					meta: {
						android_url: modelRef.value.android_url,
						ios_url: modelRef.value.ios_url,
					},
				});
				if (error) throw error;

				for (let link of links.value) {
					if (link.id === editRowRef.value.id) {
						link.url = modelRef.value.url;
						link.slug = modelRef.value.slug;
						link.meta = {
							android_url: modelRef.value.android_url,
							ios_url: modelRef.value.ios_url,
						};
					}
				}

				message.success('Link successfully updated!', { duration: messageDuration });
				showEditModal.value = false;
			} catch (error: any) {
				if (error.code == '23505') {
					message.error('Slug already exists. Please change the slug.', { duration: messageDuration });
				} else {
					message.error('Error creating link...', { duration: messageDuration });
				}
			}
		}

		const columns = [
			{
				title: 'URL',
				key: 'url',
				render(row: any) {
					return h(
						'a',
						{
							href: row.url,
							target: '_blank',
						},
						{ default: () => row.url }
					);
				},
			},
			{
				title: 'Slug',
				key: 'slug',
			},
			{
				title: 'Android URL',
				key: 'meta.android_url',
				render(row: any) {
					return h(
						'a',
						{
							href: row.meta.android_url,
							target: '_blank',
						},
						{ default: () => row.meta.android_url }
					);
				},
			},
			{
				title: 'iOS URL',
				key: 'meta.ios_url',
				render(row: any) {
					return h(
						'a',
						{
							href: row.meta.ios_url,
							target: '_blank',
						},
						{ default: () => row.meta.ios_url }
					);
				},
			},
			{
				title: 'Action',
				key: 'actions',
				render(row: any) {
					return [
						h(
							NButton,
							{
								size: 'small',
								onClick: () => handleEditLink(row),
							},
							{ default: () => 'Edit' }
						),
						h(
							NButton,
							{
								size: 'small',
								type: 'error',
								style: 'margin-left: 10px',
								onClick: () => handleDeleteLink(row),
							},
							{ default: () => 'Delete' }
						),
					];
				},
			},
		];

		const rowKey = () => {
			return 'id';
		};

		const paginationReactive = reactive({
			page: 1,
			pageSize: 10,
			showSizePicker: true,
			pageSizes: [10, 20, 30],
			onChange: (page: any) => {
				paginationReactive.page = page;
			},
			onPageSizeChange: (pageSize: any) => {
				paginationReactive.pageSize = pageSize;
				paginationReactive.page = 1;
			},
		});

		getLatestLinks();

		async function getLatestLinks() {
			try {
				const { data, error } = await fetchLinks();
				if (error) throw error;
				linksStore.updateLinks(data || []);
			} catch (error) {
				message.error('Error fetching links...', { duration: messageDuration });
			}
		}

		function query() {
			return new Promise((resolve) => {
				(async () => {
					await getLatestLinks();
					resolve({});
				})();
			});
		}

		onMounted(() => {
			query().then(() => {
				links.value = linksStore.links;
				loadingRef.value = false;
			});
		});

		function handleEditLink(row: any) {
			editRowRef.value = row;
			modelRef.value.id = row.id;
			modelRef.value.url = row.url;
			modelRef.value.slug = row.slug;
			modelRef.value.android_url = row.meta.android_url;
			modelRef.value.ios_url = row.meta.ios_url;
			showEditModal.value = true;
		}

		function handleDeleteLink(row: any) {
			dialog.warning({
				title: 'Confirm Delete Link',
				content: 'Are you sure you want to delete this link with slug of "' + row.slug + '"?',
				positiveText: 'Confirm',
				negativeText: 'Cancel',
				onPositiveClick: async () => {
					try {
						await deleteLink(row);
						linksStore.deleteLink(row);
						links.value = links.value.filter(link => link.id !== row.id);
					} catch (error) {
						message.error('Error fetching links...', { duration: messageDuration });
					}
					message.success('Link successfully deleted!');
				},
				onNegativeClick: () => {
					return;
				},
			});
		}

		return {
			tableRef,
			loadingRef,
			rowKey,
			columns,
			links,
			pagination: paginationReactive,
			showEditModal,
			model: modelRef,
			rules,
			handleGenerateSlug,
			handleEditLink,
			handleSaveEdits,
		};
	},
});
</script>

<style scoped>
.centered-view {
	width: 700px;
	margin: 0 auto;
}

.centered-form {
	width: 500px;
	margin: 0 auto;
}
</style>