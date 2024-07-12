
<template>
  <admin-view>
    <h1>Create Link</h1>
    <n-spin :show="showLoadingSpinner">
      <n-form ref="formRef" class="centered-form" :model="model" :rules="rules">
        <n-form-item path="url" label="URL">
          <n-input
            v-model:value="model.url"
            class="url-input"
            clearable
            placeholder="Enter or paste full URL"
            @update:value="handleUrlUpdate"
            @paste="handleUrlPaste"
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
        <n-row>
          <n-form-item path="start_date" label="Start Date">
            <n-date-picker
              v-model:value="model.start_date"
              type="datetime"
              placeholder="Select Start Date"
              format="yyyy-MM-dd HH:mm:ss"
            />
          </n-form-item>
          <n-form-item path="end_date" label="End Date" style="margin-left: 20px">
            <n-date-picker
              v-model:value="model.end_date"
              type="datetime"
              placeholder="Select End Date"
              format="yyyy-MM-dd HH:mm:ss"
            />
          </n-form-item>
        </n-row>
        <div style="display: flex; justify-content: center">
          <n-button round type="primary" :disabled="showLoadingSpinner" @click="handleCreateLink">
            <template #icon>
              <n-icon>
                <plus />
              </n-icon>
            </template>
            Create
          </n-button>
        </div>
      </n-form>
    </n-spin>
  </admin-view>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue';
import { addLink } from '@/services/links';
import { useAppStore } from '@/stores/appStore';
import { useLinksStore } from '@/stores/linksStore';
import {
  useMessage,
  NSpin,
  NForm,
  NFormItem,
  NInput,
  NInputGroup,
  NInputGroupLabel,
  NDatePicker,
  NIcon,
  NButton,
  NRow,
} from 'naive-ui';
import { Plus, Sync } from '@vicons/fa';
import { customAlphabet } from 'nanoid';

export default defineComponent({
  components: { NSpin, NForm, NFormItem, NInput, NInputGroup, NInputGroupLabel, NDatePicker, NIcon, NButton, NRow, Plus, Sync },
  setup() {
    const showLoadingSpinner = ref(false);
    const formRef = ref<any>(null);
    const slugRef = ref<any>(null);
    const messageDuration = 5000;
    const appStore = useAppStore();
    const linksStore = useLinksStore();
    const message = useMessage();

    const now = Date.now();
    const defaultEndDate = now + 24 * 60 * 60 * 1000;

    const modelRef = ref({
      url: '',
      slug: '',
      start_date: now,
      end_date: defaultEndDate,
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
            }
            try {
              new URL(value);
            } catch (_) {
              return new Error('Invalid URL format');
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
      start_date: [
        {
          required: true,
          validator(rule: any, value: any) {
            if (!value) {
              return new Error('Start date is required');
            }
            return true;
          },
          trigger: ['change', 'blur'],
        },
      ],
      end_date: [
        {
          required: true,
          validator(rule: any, value: any) {
            if (!value) {
              return new Error('End date is required');
            }
            return true;
          },
          trigger: ['change', 'blur'],
        },
      ],
    };

    // Remove confusion with caps I caps O and l
    const nanoid = customAlphabet('1234567890abcdefghijkmnopqrstuvwxyzABCDEFGHJKLMNPQRSTUVWXYZ', 6);

    async function handleGenerateSlug() {
      modelRef.value.slug = nanoid();
      try {
        await slugRef.value.validate();
      } catch (error) {
        return;
      }
    }

    async function handleCreateLink() {
      try {
        await formRef.value.validate();
      } catch (error) {
        return;
      }
      try {
        showLoadingSpinner.value = true;
        const startDate = new Date(modelRef.value.start_date);
        const endDate = new Date(modelRef.value.end_date);
        const { data, error } = await addLink({
          user_id: appStore.supabaseSession!.user!.id,
          url: modelRef.value.url,
          slug: modelRef.value.slug,
          start_date: startDate.toISOString(),
          end_date: endDate.toISOString(),
        });
        
        if (error) throw error;

        linksStore.addLink(data);
        resetForm();
        message.success('Link successfully created!', { duration: messageDuration });
      } catch (error: any) {
        if (error.code == '23505') {
          message.error('Slug already exists. Please change the slug.', { duration: messageDuration });
        } else {
          message.error('Error creating link...', { duration: messageDuration });
        }
      } finally {
        showLoadingSpinner.value = false;
      }
    }

    function resetForm() {
      modelRef.value.url = '';
      modelRef.value.slug = '';
      modelRef.value.start_date = Date.now();
      modelRef.value.end_date = modelRef.value.start_date + 24 * 60 * 60 * 1000;
    }

    function handleUrlUpdate(value: string) {
      modelRef.value.url = value;
    }

    function handleUrlPaste(event: ClipboardEvent) {
      const pastedText = event.clipboardData?.getData('text');
      if (pastedText) {
        event.preventDefault();
        try {
          const url = new URL(pastedText);
          modelRef.value.url = url.href;
        } catch (_) {
          // If pasted text is not a valid URL, just use it as-is
          modelRef.value.url = pastedText;
        }
      }
    }

    return {
      showLoadingSpinner,
      formRef,
      slugRef,
      model: modelRef,
      rules,
      handleGenerateSlug,
      handleCreateLink,
      handleUrlUpdate,
      handleUrlPaste,
    };
  },
});
</script>

<style scoped>
.centered-form {
  width: 500px;
  margin: 0 auto;
}

.slug-input {
  text-align: center;
}

.slug-input-inline {
  width: 33%;
  text-align: right;
}

.url-input {
  width: 100%;
}
</style>
