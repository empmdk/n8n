<script lang="ts">
import { defineComponent } from 'vue';
import type { ApiKey, IUser } from '@/Interface';
import { useToast } from '@/composables/useToast';
import { useMessage } from '@/composables/useMessage';
import { useDocumentTitle } from '@/composables/useDocumentTitle';

import CopyInput from '@/components/CopyInput.vue';
import { mapStores } from 'pinia';
import { useSettingsStore } from '@/stores/settings.store';
import { useRootStore } from '@/stores/root.store';
import { useUIStore } from '@/stores/ui.store';
import { useUsersStore } from '@/stores/users.store';
import { useCloudPlanStore } from '@/stores/cloudPlan.store';
import { DOCS_DOMAIN, MODAL_CONFIRM } from '@/constants';
import { usePageRedirectionHelper } from '@/composables/usePageRedirectionHelper';

export default defineComponent({
	name: 'SettingsApiView',
	components: {
		CopyInput,
	},
	setup() {
		return {
			...useToast(),
			...useMessage(),
			...useUIStore(),
			pageRedirectionHelper: usePageRedirectionHelper(),
			documentTitle: useDocumentTitle(),
		};
	},
	data() {
		return {
			loading: false,
			mounted: false,
			apiKeys: [] as ApiKey[],
			swaggerUIEnabled: false,
			apiDocsURL: '',
		};
	},
	mounted() {
		this.documentTitle.set(this.$locale.baseText('settings.api'));
		if (!this.isPublicApiEnabled) return;

		void this.getApiKeys();
		const baseUrl = this.rootStore.baseUrl;
		const apiPath = this.settingsStore.publicApiPath;
		const latestVersion = this.settingsStore.publicApiLatestVersion;
		this.swaggerUIEnabled = this.settingsStore.isSwaggerUIEnabled;
		this.apiDocsURL = this.swaggerUIEnabled
			? `${baseUrl}${apiPath}/v${latestVersion}/docs`
			: `https://${DOCS_DOMAIN}/api/api-reference/`;
	},
	computed: {
		...mapStores(useRootStore, useSettingsStore, useUsersStore, useCloudPlanStore, useUIStore),
		currentUser(): IUser | null {
			return this.usersStore.currentUser;
		},
		isTrialing(): boolean {
			return this.cloudPlanStore.userIsTrialing;
		},
		isLoadingCloudPlans(): boolean {
			return this.cloudPlanStore.state.loadingPlan;
		},
		isPublicApiEnabled(): boolean {
			return this.settingsStore.isPublicApiEnabled;
		},
		isRedactedApiKey(): boolean {
			if (!this.apiKeys) return false;
			return this.apiKeys[0].apiKey.includes('*');
		},
	},
	methods: {
		onUpgrade() {
			void this.pageRedirectionHelper.goToUpgrade('settings-n8n-api', 'upgrade-api', 'redirect');
		},
		async showDeleteModal() {
			const confirmed = await this.confirm(
				this.$locale.baseText('settings.api.delete.description'),
				this.$locale.baseText('settings.api.delete.title'),
				{
					confirmButtonText: this.$locale.baseText('settings.api.delete.button'),
					cancelButtonText: this.$locale.baseText('generic.cancel'),
				},
			);
			if (confirmed === MODAL_CONFIRM) {
				await this.deleteApiKey();
			}
		},
		async getApiKeys() {
			try {
				this.apiKeys = await this.settingsStore.getApiKeys();
			} catch (error) {
				this.showError(error, this.$locale.baseText('settings.api.view.error'));
			} finally {
				this.mounted = true;
			}
		},
		async createApiKey() {
			this.loading = true;

			try {
				const newApiKey = await this.settingsStore.createApiKey();
				this.apiKeys.push(newApiKey);
			} catch (error) {
				this.showError(error, this.$locale.baseText('settings.api.create.error'));
			} finally {
				this.loading = false;
				this.$telemetry.track('User clicked create API key button');
			}
		},
		async deleteApiKey() {
			try {
				await this.settingsStore.deleteApiKey(this.apiKeys[0].id);
				this.showMessage({
					title: this.$locale.baseText('settings.api.delete.toast'),
					type: 'success',
				});
				this.apiKeys = [];
			} catch (error) {
				this.showError(error, this.$locale.baseText('settings.api.delete.error'));
			} finally {
				this.$telemetry.track('User clicked delete API key button');
			}
		},
		onCopy() {
			this.$telemetry.track('User clicked copy API key button');
		},
	},
});
</script>

<template>
	<div :class="$style.container">
		<div :class="$style.header">
			<n8n-heading size="2xlarge">
				{{ $locale.baseText('settings.api') }}
				<span :style="{ fontSize: 'var(--font-size-s)', color: 'var(--color-text-light)' }">
					({{ $locale.baseText('generic.beta') }})
				</span>
			</n8n-heading>
		</div>

		<div v-if="apiKeys.length">
			<p class="mb-s">
				<n8n-info-tip :bold="false">
					<i18n-t keypath="settings.api.view.info" tag="span">
						<template #apiAction>
							<a
								href="https://docs.n8n.io/api"
								target="_blank"
								v-text="$locale.baseText('settings.api.view.info.api')"
							/>
						</template>
						<template #webhookAction>
							<a
								href="https://docs.n8n.io/integrations/core-nodes/n8n-nodes-base.webhook/"
								target="_blank"
								v-text="$locale.baseText('settings.api.view.info.webhook')"
							/>
						</template>
					</i18n-t>
				</n8n-info-tip>
			</p>
			<n8n-card class="mb-4xs" :class="$style.card">
				<span :class="$style.delete">
					<n8n-link :bold="true" @click="showDeleteModal">
						{{ $locale.baseText('generic.delete') }}
					</n8n-link>
				</span>

				<div>
					<CopyInput
						:label="apiKeys[0].label"
						:value="apiKeys[0].apiKey"
						:copy-button-text="$locale.baseText('generic.clickToCopy')"
						:toast-title="$locale.baseText('settings.api.view.copy.toast')"
						:redact-value="true"
						:disable-copy="isRedactedApiKey"
						:hint="!isRedactedApiKey ? $locale.baseText('settings.api.view.copy') : ''"
						@copy="onCopy"
					/>
				</div>
			</n8n-card>
			<div :class="$style.hint">
				<n8n-text size="small">
					{{
						$locale.baseText(`settings.api.view.${swaggerUIEnabled ? 'tryapi' : 'more-details'}`)
					}}
				</n8n-text>
				{{ ' ' }}
				<n8n-link :to="apiDocsURL" :new-window="true" size="small">
					{{
						$locale.baseText(
							`settings.api.view.${swaggerUIEnabled ? 'apiPlayground' : 'external-docs'}`,
						)
					}}
				</n8n-link>
			</div>
		</div>
		<n8n-action-box
			v-else-if="!isPublicApiEnabled && isTrialing"
			data-test-id="public-api-upgrade-cta"
			:heading="$locale.baseText('settings.api.trial.upgradePlan.title')"
			:description="$locale.baseText('settings.api.trial.upgradePlan.description')"
			:button-text="$locale.baseText('settings.api.trial.upgradePlan.cta')"
			@click:button="onUpgrade"
		/>
		<n8n-action-box
			v-else-if="mounted && !isLoadingCloudPlans"
			:button-text="
				$locale.baseText(
					loading ? 'settings.api.create.button.loading' : 'settings.api.create.button',
				)
			"
			:description="$locale.baseText('settings.api.create.description')"
			@click:button="createApiKey"
		/>
	</div>
</template>

<style lang="scss" module>
.container {
	> * {
		margin-bottom: var(--spacing-2xl);
	}
}

.header {
	display: flex;
	align-items: center;
	white-space: nowrap;

	*:first-child {
		flex-grow: 1;
	}
}

.card {
	position: relative;
}

.delete {
	position: absolute;
	display: inline-block;
	top: var(--spacing-s);
	right: var(--spacing-s);
}

.hint {
	color: var(--color-text-light);
}
</style>
