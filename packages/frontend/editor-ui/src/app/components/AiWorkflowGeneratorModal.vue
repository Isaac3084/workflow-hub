<script setup lang="ts">
import { ref } from 'vue';

const emit = defineEmits<{ close: []; import: [json: string] }>();

const userPrompt = ref('');
const isGenerating = ref(false);
const error = ref('');
const generatedJson = ref('');
const step = ref<'input' | 'preview'>('input');

const examples = [
  'Every day at 9 AM, fetch top tech news and send me a summary email',
  'When I receive a support ticket, classify its urgency using AI and route it',
  'Monitor a website every hour and notify me on Telegram if it goes down',
  'Read CSV files from a folder and save structured data to a spreadsheet',
];

async function generate() {
  if (!userPrompt.value.trim()) return;

  isGenerating.value = true;
  error.value = '';
  generatedJson.value = '';

  const systemPrompt = `You are an n8n workflow builder. Generate a valid n8n workflow JSON for the following task.

RULES:
- Return ONLY the raw JSON object, no markdown, no explanation
- Use real n8n node types like: n8n-nodes-base.manualTrigger, n8n-nodes-base.scheduleTrigger, n8n-nodes-base.httpRequest, n8n-nodes-base.set, n8n-nodes-base.if, n8n-nodes-base.code
- For AI tasks use @n8n/n8n-nodes-langchain.ollama
- Every node needs: parameters, id (short unique string), name, type, typeVersion (1), position [x, y]
- Connections map source node names to target node names
- Use positions spaced 220px apart horizontally starting at [100, 300]

OUTPUT FORMAT EXAMPLE:
{"name":"My Workflow","nodes":[{"parameters":{},"id":"t1","name":"Start","type":"n8n-nodes-base.manualTrigger","typeVersion":1,"position":[100,300]}],"connections":{"Start":{"main":[[{"node":"Next Node","type":"main","index":0}]]}}}

TASK: ${userPrompt.value}`;

  try {
    const res = await fetch('http://127.0.0.1:11434/api/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'llama3.1:8b',
        prompt: systemPrompt,
        stream: false,
      }),
    });

    if (!res.ok) throw new Error('Ollama is offline. Run: ollama serve');

    const data = await res.json();
    let raw = data.response.replace(/```json/gi, '').replace(/```/g, '').trim();

    // Extract JSON object from the response
    const match = raw.match(/\{[\s\S]*\}/);
    if (!match) throw new Error('AI did not return valid JSON. Try a simpler description.');

    // Validate it parses
    JSON.parse(match[0]);
    generatedJson.value = match[0];
    step.value = 'preview';
  } catch (e: unknown) {
    error.value = e instanceof Error ? e.message : 'Unknown error occurred';
  } finally {
    isGenerating.value = false;
  }
}

function importWorkflow() {
  emit('import', generatedJson.value);
  emit('close');
}

function tryExample(example: string) {
  userPrompt.value = example;
}

function reset() {
  step.value = 'input';
  generatedJson.value = '';
  error.value = '';
}

let parsedWorkflow = { name: '', nodes: [] as { name: string; type: string }[] };
try {
  if (generatedJson.value) parsedWorkflow = JSON.parse(generatedJson.value);
} catch {}
</script>

<template>
  <div :class="$style.overlay" @click.self="$emit('close')">
    <div :class="$style.modal">
      <!-- Header -->
      <div :class="$style.header">
        <div :class="$style.headerLeft">
          <span :class="$style.headerIcon">🧠</span>
          <div>
            <div :class="$style.headerTitle">AI Workflow Generator</div>
            <div :class="$style.headerSub">Describe what you want → AI builds it</div>
          </div>
        </div>
        <button :class="$style.closeBtn" @click="$emit('close')">✕</button>
      </div>

      <!-- Input Step -->
      <div v-if="step === 'input'" :class="$style.body">
        <textarea
          v-model="userPrompt"
          :class="$style.textarea"
          placeholder="Describe your workflow in plain English...
e.g. 'Every morning at 8 AM, fetch the top 5 news headlines and send them to my email'"
          rows="4"
          :disabled="isGenerating"
        />

        <!-- Error -->
        <div v-if="error" :class="$style.error">⚠️ {{ error }}</div>

        <!-- Examples -->
        <div :class="$style.examplesSection">
          <div :class="$style.examplesLabel">💡 Try an example:</div>
          <div :class="$style.examples">
            <button
              v-for="ex in examples"
              :key="ex"
              :class="$style.exampleBtn"
              @click="tryExample(ex)"
            >
              {{ ex }}
            </button>
          </div>
        </div>

        <!-- Actions -->
        <div :class="$style.actions">
          <button :class="$style.cancelBtn" @click="$emit('close')">Cancel</button>
          <button
            :class="[$style.generateBtn, isGenerating && $style.generating]"
            :disabled="!userPrompt.trim() || isGenerating"
            @click="generate"
          >
            <span v-if="isGenerating">⏳ Generating with Ollama...</span>
            <span v-else>⚡ Generate Workflow</span>
          </button>
        </div>
      </div>

      <!-- Preview Step -->
      <div v-else :class="$style.body">
        <div :class="$style.successBanner">
          ✅ Workflow generated successfully!
        </div>

        <div :class="$style.previewInfo">
          <strong>Name:</strong>
          {{ (() => { try { return JSON.parse(generatedJson).name } catch { return 'Generated Workflow' } })() }}
          &nbsp;|&nbsp;
          <strong>Nodes:</strong>
          {{ (() => { try { return JSON.parse(generatedJson).nodes?.length } catch { return '?' } })() }}
        </div>

        <pre :class="$style.jsonPreview">{{ (() => { try { return JSON.stringify(JSON.parse(generatedJson), null, 2).substring(0, 800) + '...' } catch { return generatedJson.substring(0, 800) } })() }}</pre>

        <div :class="$style.actions">
          <button :class="$style.cancelBtn" @click="reset">← Try Again</button>
          <button :class="$style.generateBtn" @click="importWorkflow">
            ✅ Import Workflow
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" module>
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  backdrop-filter: blur(3px);
}

.modal {
  background: white;
  border-radius: 16px;
  width: 640px;
  max-width: 95vw;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.2);
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem 1.75rem 1rem;
  border-bottom: 1px solid #f0f0f0;
}

.headerLeft { display: flex; align-items: center; gap: 0.75rem; }
.headerIcon { font-size: 2rem; }
.headerTitle { font-size: 1.1rem; font-weight: 700; color: #1a1a2e; }
.headerSub { font-size: 0.8rem; color: #888; }

.closeBtn {
  background: none;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  width: 32px;
  height: 32px;
  cursor: pointer;
  color: #888;
  font-size: 0.9rem;
  &:hover { background: #f5f5f5; color: #333; }
}

.body { padding: 1.5rem 1.75rem; }

.textarea {
  width: 100%;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 0.875rem;
  font-size: 0.95rem;
  line-height: 1.5;
  resize: vertical;
  outline: none;
  font-family: inherit;
  color: #333;
  box-sizing: border-box;
  &:focus { border-color: #6366f1; box-shadow: 0 0 0 3px rgba(99,102,241,0.1); }
  &:disabled { background: #fafafa; }
}

.error {
  margin-top: 0.75rem;
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #dc2626;
  font-size: 0.88rem;
}

.examplesSection { margin-top: 1.25rem; }
.examplesLabel { font-size: 0.82rem; color: #888; margin-bottom: 0.5rem; font-weight: 500; }
.examples { display: flex; flex-direction: column; gap: 0.4rem; }

.exampleBtn {
  text-align: left;
  background: #f8f9fa;
  border: 1px solid #e5e7eb;
  border-radius: 7px;
  padding: 0.5rem 0.75rem;
  font-size: 0.83rem;
  color: #555;
  cursor: pointer;
  transition: all 0.15s;
  &:hover { background: #ede9fe; border-color: #a5b4fc; color: #4f46e5; }
}

.actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
  margin-top: 1.5rem;
}

.cancelBtn {
  background: none;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 0.625rem 1.25rem;
  cursor: pointer;
  font-size: 0.9rem;
  color: #555;
  &:hover { background: #f5f5f5; }
}

.generateBtn {
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.5rem;
  cursor: pointer;
  font-size: 0.9rem;
  font-weight: 600;
  transition: opacity 0.15s;
  &:disabled { opacity: 0.5; cursor: not-allowed; }
  &:not(:disabled):hover { opacity: 0.9; }
}

.generating { opacity: 0.8; }

.successBanner {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  color: #15803d;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  font-weight: 600;
  margin-bottom: 1rem;
}

.previewInfo {
  font-size: 0.88rem;
  color: #555;
  margin-bottom: 0.75rem;
}

.jsonPreview {
  background: #1e1e2e;
  color: #a6e3a1;
  border-radius: 10px;
  padding: 1rem;
  font-size: 0.78rem;
  overflow: auto;
  max-height: 220px;
  font-family: 'Consolas', monospace;
  white-space: pre-wrap;
  word-break: break-all;
}
</style>
