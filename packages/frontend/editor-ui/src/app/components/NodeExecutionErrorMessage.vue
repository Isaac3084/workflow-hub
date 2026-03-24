<script lang="ts" setup>
import { ref } from 'vue';
import { useI18n } from '@n8n/i18n';

const i18n = useI18n();
const props = defineProps<{ nodeName: string; errorMessage?: string }>();

const explaining = ref(false);
const explanation = ref('');
const showExplanation = ref(false);

async function explainError() {
  if (!props.errorMessage) return;
  explaining.value = true;
  explanation.value = '';
  showExplanation.value = true;

  try {
    const res = await fetch('http://127.0.0.1:11434/api/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'llama3.1:8b',
        prompt: `You are an expert n8n workflow debugger. A workflow node called "${props.nodeName}" has failed with this error:\n\n"${props.errorMessage}"\n\nExplain in 2-3 simple sentences:\n1. What caused this error\n2. How to fix it\n\nKeep it short and practical. No technical jargon.`,
        stream: false,
      }),
    });
    const data = await res.json();
    explanation.value = data.response?.trim() || 'Could not get explanation.';
  } catch {
    explanation.value = '❌ Ollama is offline. Start it with: ollama serve';
  } finally {
    explaining.value = false;
  }
}
</script>

<template>
  <div>
    <span v-n8n-html="errorMessage" data-test-id="sanitized-error-message" />
    <br />
    <a data-action="openNodeDetail" :data-action-parameter-node="nodeName">{{
      i18n.baseText('node.executionError.openNode')
    }}</a>

    <!-- AI Error Explainer -->
    <div style="margin-top: 8px;">
      <button
        v-if="!showExplanation"
        style="
          background: linear-gradient(135deg,#6366f1,#8b5cf6);
          color: white;
          border: none;
          border-radius: 6px;
          padding: 4px 10px;
          font-size: 11px;
          cursor: pointer;
          font-weight: 600;
        "
        @click="explainError"
      >
        🤖 Explain this error
      </button>
      <div
        v-if="showExplanation"
        style="
          margin-top: 8px;
          background: #1e1e2e;
          border-radius: 8px;
          padding: 10px 12px;
          font-size: 12px;
          line-height: 1.5;
        "
      >
        <div style="color:#a6e3a1; font-weight:600; margin-bottom:4px;">🧠 AI Explanation</div>
        <div v-if="explaining" style="color:#cdd6f4;">⏳ Asking Ollama AI...</div>
        <div v-else style="color:#cdd6f4; white-space:pre-wrap;">{{ explanation }}</div>
        <button
          style="background:none;border:none;color:#888;cursor:pointer;font-size:11px;margin-top:6px;"
          @click="showExplanation = false; explanation = '';"
        >✕ Close</button>
      </div>
    </div>
  </div>
</template>
