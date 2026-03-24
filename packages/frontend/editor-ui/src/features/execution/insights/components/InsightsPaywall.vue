<script lang="ts" setup>
import { useI18n } from '@n8n/i18n';
import { usePageRedirectionHelper } from '@/app/composables/usePageRedirectionHelper';

import { N8nButton, N8nIcon, N8nText } from '@n8n/design-system';
const pageRedirectionHelper = usePageRedirectionHelper();
const i18n = useI18n();

const goToUpgrade = async () => {
	await pageRedirectionHelper.goToUpgrade('insights', 'upgrade-insights');
};
</script>

<template>
	<div :class="$style.callout" style="display: none;">
		<!-- Upgrade prompt removed for Academic Project -->
	</div>
</template>

<style lang="scss" module>
.callout {
	display: flex;
	flex-direction: column;
	height: 100%;
	padding: 6rem 0;
	align-items: center;
	max-width: 360px;
	margin: 0 auto;
	text-align: center;
	gap: 10px;
	justify-content: center;
}
</style>
