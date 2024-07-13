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
			:closable="false"
			:mask-closable="false"
		>
			<template #header>
				<div>Edit Link</div>
			</template>
			<div>
				<n-spin :show="showLoadingSpinner">
					<n-form ref="formRef" class="centered-form" :model="model" :rules="rules">
						<n-form-item path="url" label="URL">
							<n-input
								v-model:value="model.url_raw"
								class="url-input"
								pair
								clearable
								separator="://"
								:placeholder="['Protocol', 'Web Address']"
								@change="handleUrlUpdate"
								@update:value="handleUrlUpdate"
							></n-input>
						</n-form-item>
						<n-row>
							<n-form-item ref="slugRef" path="slug" label="Slug" style="flex-grow: 1">
								<n-input-group>
									<n-input-group-label class="slug-input-inline">/</n-input-group-label>
									<n-input v-model:value="model.slug" class="slug-input" placeholder="Enter Slug" />
								</n-input-group>
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
							<n-input
								v-model:value="model.android_url_raw"
								class="url-input"
								pair
								clearable
								separator="://"
								:placeholder="['Protocol', 'Web Address']"
								@change="handleAndroidUrlUpdate"
								@update:value="handleAndroidUrlUpdate"
							></n-input>
						</n-form-item>
						<n-form-item path="ios_url" label="iOS URL" style="flex-grow: 1">
							<n-input
								v-model:value="model.ios_url_raw"
								class="url-input"
								pair
								clearable
								separator="://"
								:placeholder="['Protocol', 'Web Address']"
								@change="handleIosUrlUpdate"
								@update:value="handleIosUrlUpdate"
							></n-input>
						</n-form-item>
						<n-form-item path="end_date" label="End Date">
							<n-date-picker
								v-model:value="model.end_date"
								type="datetime"
								clearable
								:is-date-disabled="isDateDisabled"
							/>
						</n-form-item>
					</n-form>
				</n-spin>
			</div>
			<template #footer>
				<div>
					<n-button type="primary" :disabled="showLoadingSpinner" @click="handleSaveEdits">Update</n-button>
					<n-button style="margin-left: 20px" :disabled="showLoadingSpinner" @click="showEditModal = false">Cancel</n-button>
				</div>
			</template>
		</n-modal>
	</admin-view>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, onMounted, h, computed } from 'vue';
import { fetchLinks, editLink, deleteLink } from '@/services/links';
import {
	useMessage,
	useDialog,
	NDataTable,
	NButton,
	NModal,
	NSpin,
	NForm,
	NFormItem,
	NInput,
	NInputGroup,
	NInputGroupLabel,
	NIcon,
	NRow,
	NDatePicker,
} from 'naive-ui';
import { Sync } from '@vicons/fa';
import { useLinksStore } from '@/stores/linksStore';
import { Link } from '@/types/global';
import { customAlphabet } from 'nanoid';

export default defineComponent({
	components: {
		NDataTable,
		NModal,
		NSpin,
		NButton,
		NForm,
		NFormItem,
		NInput,
		NInputGroup,
		NInputGroupLabel,
		NIcon,
		NRow,
		NDatePicker,
		Sync,
	},
	setup() {
		const messageDuration = 5000;
		const linksStore = useLinksStore();
		const message = useMessage();
		const dialog = useDialog();
		const formRef = ref();
		const tableRef = ref();
		const loadingRef = ref(true);
		const links = ref<Link[]>([]);
		const showEditModal = ref(false);
		const showLoadingSpinner = ref(false);

		const modelRef = ref({
			id: '',
			url: computed(() => {
				if (!modelRef.value.url_raw[0] && !modelRef.value.url_raw[1]) return '';
				return modelRef.value.url_raw[0] + '://' + modelRef.value.url_raw[1];
			}),
			url_raw: ['', ''],
			slug: '',
			android_url: computed(() => {
				if (!modelRef.value.android_url_raw[0] && !modelRef.value.android_url_raw[1]) return '';
				return modelRef.value.android_url_raw[0] + '://' + modelRef.value.android_url_raw[1];
			}),
			android_url_raw: ['', ''],
			ios_url: computed(() => {
				if (!modelRef.value.ios_url_raw[0] && !modelRef.value.ios_url_raw[1]) return '';
				return modelRef.value.ios_url_raw[0] + '://' + modelRef.value.ios_url_raw[1];
			}),
			ios_url_raw: ['', ''],
			end_date: null as number | null,
		});

		const rules = {
			url: [
				{
					required: true,
					validator(rule: any, value: any) {
						if (!value) {
							return new Error('URL is required');
						} else if (value.length > 2083) {
							return new Error('URL has to be 2083 characters or below.');
						} else if (String(value).startsWith('://')) {
							return new Error('Please enter a protocol.');
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
						if (value.length > 50) {
							return new Error('Slug has to be 50 characters or below.');
						}
						return true;
					},
					trigger: ['input', 'blur'],
				},
			],
			android_url: [
				{
					validator(rule: any, value: any) {
						if (!value) {
							return true;
						}
						if (value.length > 2083) {
							return new Error('Android URL has to be 2083 characters or below.');
						} else if (String(value).startsWith('://')) {
							return new Error('Please enter a protocol.');
						}
						return true;
					},
					trigger: ['input', 'blur'],
				},
			],
			ios_url: [
				{
					validator(rule: any, value: any) {
						if (!value) {
							return true;
						}
						if (value.length > 2083) {
							return new Error('iOS URL has to be 2083 characters or below.');
						} else if (String(value).startsWith('://')) {
							return new Error('Please enter a protocol.');
						}
						return true;
					},
					trigger: ['input', 'blur'],
				},
			],
		};

		const nanoid = customAlphabet('1234567890abcdefghijkmnopqrstuvwxyzABCDEFGHJKLMNPQRSTUVWXYZ', 6);

		function calculateTimeLeft(endDate: string | undefined): string {
			if (!endDate) {
				return 'No expiration';
			}

			const now = new Date();
			const expirationDate = new Date(endDate);
			const timeDiff = expirationDate.getTime() - now.getTime();
			
			if (timeDiff <= 0) {
				return 'Expired';
			}

			const days = Math.floor(timeDiff / (1000 * 60 * 60 * 24));
			const hours = Math.floor((timeDiff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));

			return `${days}d ${hours}h`;
		}

		const columns: any = [
			{
				title: 'URL',
				key: 'url',
				render(row: Link) {
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
				title: 'ShortURL',
				key: 'slug',
				render(row: Link) {
					const fullUrl = `https://supaflare-worker.studyyymusic.workers.dev/${row.slug}`;
					return h(
						'a',
						{
							href: fullUrl,
							target: '_blank',
						},
						{ default: () => fullUrl }
					);
				},
			},
			{
				title: 'TTL',
				key: 'ttl',
				render(row: Link) {
					return h('span', {}, { default: () => calculateTimeLeft(row.end_date) });
				},
			},
			{
				title: 'Action',
				key: 'actions',
				width: '150px',
				render(row: Link) {
					return h('div', [
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
					]);
				},
			},
		];

		const rowKey = (row: Link) => row.id || '';

		const paginationReactive = reactive({
			page: 1,
			pageSize: 10,
			showSizePicker: true,
			pageSizes: [10, 20, 30],
			onChange: (page: number) => {
				paginationReactive.page = page;
			},
			onPageSizeChange: (pageSize: number) => {
				paginationReactive.pageSize = pageSize;
				paginationReactive.page = 1;
			},
		});

		onMounted(() => {
			getLatestLinks();
		});

		async function getLatestLinks() {
			try {
				loadingRef.value = true;
				const { data, error } = await fetchLinks();
				if (error) throw error;
				linksStore.updateLinks(data || []);
				links.value = linksStore.links;
			} catch (error) {
				message.error('Error fetching links...', { duration: messageDuration });
			} finally {
				loadingRef.value = false;
			}
		}

		function handleGenerateSlug() {
			modelRef.value.slug = nanoid();
		}

		async function handleSaveEdits() {
			try {
				await formRef.value?.validate();
			} catch (error) {
				return;
			}
			try {
				showLoadingSpinner.value = true;
				const { error } = await editLink(modelRef.value.id, {
					url: modelRef.value.url,
					slug: modelRef.value.slug,
					meta: {
						android_url: modelRef.value.android_url,
						ios_url: modelRef.value.ios_url,
					},
					end_date: modelRef.value.end_date ? new Date(modelRef.value.end_date).toISOString() : null,
				});
				if (error) throw error;

				await getLatestLinks();

				message.success('Link successfully updated!', { duration: messageDuration });
				showEditModal.value = false;
			} catch (error: any) {
				if (error.code == '23505') {
					message.error('Slug already exists. Please change the slug.', { duration: messageDuration });
				} else {
					message.error('Error updating link...', { duration: messageDuration });
				}
			} finally {
				showLoadingSpinner.value = false;
			}
		}

		function handleEditLink(row: Link) {
			modelRef.value.id = row.id || '';
			modelRef.value.slug = row.slug;

			if (row.url) {
				const fields = String(row.url).split('://');
				modelRef.value.url_raw = [fields[0], fields.slice(1).join('://')];
			} else {
				modelRef.value.url_raw = ['', ''];
			}
			if (row.meta?.android_url) {
				const fields = String(row.meta.android_url).split('://');
				modelRef.value.android_url_raw = [fields[0], fields.slice(1).join('://')];
			} else {
				modelRef.value.android_url_raw = ['', ''];
			}
			if (row.meta?.ios_url) {
				const fields = String(row.meta.ios_url).split('://');
				modelRef.value.ios_url_raw = [fields[0], fields.slice(1).join('://')];
			} else {
				modelRef.value.ios_url_raw = ['', ''];
			}
			modelRef.value.end_date = row.end_date ? new Date(row.end_date).getTime() : null;

			showEditModal.value = true;
		}

 	        async function performDeleteLink(row: Link) {
			try {
				loadingRef.value = true;
				await deleteLink(row.id);
				linksStore.deleteLink(row);
				links.value = links.value.filter((link) => link.id !== row.id);
				message.success('Link successfully deleted!');
			} catch (error) {
				message.error('Error deleting link...', { duration: messageDuration });
			} finally {
				loadingRef.value = false;
			}
		}

		function handleUrlUpdate(val: [string, string]) {
			if (String(val[0]).includes('://')) {
				const splits = String(val[0]).split('://');
				if (splits.length > 1) {
					modelRef.value.url_raw[0] = splits[0];
					modelRef.value.url_raw[1] = splits.slice(1).join('://');
				}
			} else if (String(val[1]).includes('://')) {
				const splits = String(val[1]).split('://');
				if (splits.length > 1) {
					if (!val[0] || val[0] === splits[0]) {
						modelRef.value.url_raw[0] = splits[0];
						modelRef.value.url_raw[1] = splits.slice(1).join('://');
					}
				}
			}
		}

		function handleAndroidUrlUpdate(val: [string, string]) {
			if (String(val[0]).includes('://')) {
				const splits = String(val[0]).split('://');
				if (splits.length > 1) {
					modelRef.value.android_url_raw[0] = splits[0];
					modelRef.value.android_url_raw[1] = splits.slice(1).join('://');
				}
			} else if (String(val[1]).includes('://')) {
				const splits = String(val[1]).split('://');
				if (splits.length > 1) {
					if (!val[0] || val[0] === splits[0]) {
						modelRef.value.android_url_raw[0] = splits[0];
						modelRef.value.android_url_raw[1] = splits.slice(1).join('://');
					}
				}
			}
		}

		function handleIosUrlUpdate(val: [string, string]) {
			if (String(val[0]).includes('://')) {
				const splits = String(val[0]).split('://');
				if (splits.length > 1) {
					modelRef.value.ios_url_raw[0] = splits[0];
					modelRef.value.ios_url_raw[1] = splits.slice(1).join('://');
				}
			} else if (String(val[1]).includes('://')) {
				const splits = String(val[1]).split('://');
				if (splits.length > 1) {
					if (!val[0] || val[0] === splits[0]) {
						modelRef.value.ios_url_raw[0] = splits[0];
						modelRef.value.ios_url_raw[1] = splits.slice(1).join('://');
					}
				}
			}
		}

		function isDateDisabled(timestamp: number) {
			return timestamp < Date.now();
		}

		return {
			formRef,
			tableRef,
			loadingRef,
			rowKey,
			columns,
			links,
			pagination: paginationReactive,
			showEditModal,
			model: modelRef,
			rules,
			showLoadingSpinner,
			handleGenerateSlug,
			handleEditLink,
			handleSaveEdits,
			handleUrlUpdate,
			handleAndroidUrlUpdate,
			handleIosUrlUpdate,
			isDateDisabled,
		};
	},
});
</script>

<style scoped>
.centered-view {
	margin: 0 auto;
}

.centered-form {
	width: 500px;
	margin: 0 auto;
}

.slug-input {
	text-align: center;
}

.slug-input-inline {
	width: '33%';
	text-align: 'right';
}

.url-input :deep(.n-input-wrapper):first-child {
	flex-grow: 0;
	width: 80px;
}

.url-input :deep(.n-input-wrapper):first-child input {
	text-align: right;
}

.url-input :deep(.n-input-wrapper):nth-child(3) input {
	text-align: left;
}
</style>
