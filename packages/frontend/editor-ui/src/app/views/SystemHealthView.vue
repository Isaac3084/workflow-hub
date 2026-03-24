<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue';

interface ModelInfo {
  name: string;
  size: number;
  details?: { parameter_size?: string; quantization_level?: string };
}

interface RunningModel {
  name: string;
  size_vram?: number;
}

const ollamaOnline = ref(false);
const ollamaModels = ref<ModelInfo[]>([]);
const activeModel = ref('');
const vramUsed = ref(0);
const vramTotal = 6;
const workflowCount = ref(0);
const lastUpdated = ref('');
const refreshing = ref(false);

let intervalId: ReturnType<typeof setInterval>;

const vramPercent = computed(() => Math.min((vramUsed.value / vramTotal) * 100, 100));

function formatBytes(bytes: number): string {
  if (!bytes) return '—';
  return `${(bytes / 1024 ** 3).toFixed(1)} GB`;
}

async function fetchData() {
  refreshing.value = true;
  try {
    const [tagsRes, psRes, wfRes] = await Promise.all([
      fetch('http://127.0.0.1:11434/api/tags').catch(() => null),
      fetch('http://127.0.0.1:11434/api/ps').catch(() => null),
      fetch('/api/v1/workflows?limit=1', { credentials: 'include' }).catch(() => null),
    ]);

    ollamaOnline.value = !!(tagsRes && tagsRes.ok);

    if (tagsRes && tagsRes.ok) {
      const d = await tagsRes.json();
      ollamaModels.value = d.models || [];
    }

    if (psRes && psRes.ok) {
      const d = await psRes.json();
      const running: RunningModel[] = d.models || [];
      if (running.length > 0) {
        activeModel.value = running[0].name;
        vramUsed.value = running[0].size_vram
          ? parseFloat((running[0].size_vram / 1024 ** 3).toFixed(1))
          : 0;
      } else {
        activeModel.value = '';
        vramUsed.value = 0;
      }
    }

    if (wfRes && wfRes.ok) {
      const d = await wfRes.json();
      workflowCount.value = d.count || 0;
    }
  } finally {
    refreshing.value = false;
    lastUpdated.value = new Date().toLocaleTimeString();
  }
}

onMounted(() => {
  void fetchData();
  intervalId = setInterval(() => void fetchData(), 5000);
});
onUnmounted(() => clearInterval(intervalId));
</script>

<template>
  <div :class="$style.page">
    <!-- Page Header -->
    <div :class="$style.pageHeader">
      <div :class="$style.pageTitle">
        <span :class="$style.pageTitleIcon">⚡</span>
        <div>
          <h1 :class="$style.h1">System Health</h1>
          <p :class="$style.subtitle">Real-time monitoring for Workflow Hub</p>
        </div>
      </div>
      <div :class="$style.headerRight">
        <span v-if="lastUpdated" :class="$style.lastUpdated">Updated {{ lastUpdated }}</span>
        <button :class="[$style.refreshBtn, refreshing && $style.refreshing]" @click="fetchData">
          <span :class="$style.refreshIcon">↻</span> Refresh
        </button>
      </div>
    </div>

    <!-- Status Cards -->
    <div :class="$style.cardGrid">
      <!-- Ollama Status -->
      <div :class="[$style.card, ollamaOnline ? $style.cardSuccess : $style.cardDanger]">
        <div :class="$style.cardGlow" />
        <div :class="$style.cardTop">
          <span :class="$style.cardEmoji">🤖</span>
          <span :class="[$style.badge, ollamaOnline ? $style.badgeGreen : $style.badgeRed]">
            <span :class="$style.badgeDot" />{{ ollamaOnline ? 'Online' : 'Offline' }}
          </span>
        </div>
        <div :class="$style.cardLabel">Ollama AI Engine</div>
        <div :class="$style.cardValue">{{ ollamaOnline ? 'Connected' : 'Not Running' }}</div>
        <div :class="$style.cardSub">
          {{ activeModel || (ollamaOnline ? 'No model active' : 'Run: ollama serve') }}
        </div>
      </div>

      <!-- VRAM -->
      <div :class="[$style.card, $style.cardIndigo]">
        <div :class="$style.cardGlow" />
        <div :class="$style.cardTop">
          <span :class="$style.cardEmoji">🎮</span>
          <span :class="$style.cardTopRight">{{ vramUsed > 0 ? `${vramUsed} GB` : '—' }} / {{ vramTotal }} GB</span>
        </div>
        <div :class="$style.cardLabel">GPU VRAM Usage</div>
        <div :class="$style.cardValue">{{ vramPercent.toFixed(0) }}%</div>
        <div :class="$style.progressTrack">
          <div :class="$style.progressFill" :style="{ width: `${vramPercent}%` }" />
        </div>
        <div :class="$style.cardSub">6GB VRAM · llama3.1:8b</div>
      </div>

      <!-- Workflows -->
      <div :class="[$style.card, $style.cardPurple]">
        <div :class="$style.cardGlow" />
        <div :class="$style.cardTop">
          <span :class="$style.cardEmoji">⚙️</span>
        </div>
        <div :class="$style.cardLabel">Workflows Saved</div>
        <div :class="[$style.cardValue, $style.cardValueLarge]">{{ workflowCount }}</div>
        <div :class="$style.cardSub">Stored in local SQLite</div>
      </div>

      <!-- Platform -->
      <div :class="[$style.card, $style.cardTeal]">
        <div :class="$style.cardGlow" />
        <div :class="$style.cardTop">
          <span :class="$style.cardEmoji">🌐</span>
          <span :class="[$style.badge, $style.badgeGreen]">
            <span :class="$style.badgeDot" />Live
          </span>
        </div>
        <div :class="$style.cardLabel">Workflow Hub</div>
        <div :class="$style.cardValue">Running</div>
        <div :class="$style.cardSub">localhost:5678 · 16GB RAM</div>
      </div>
    </div>

    <!-- Models Table -->
    <div :class="$style.section">
      <div :class="$style.sectionHeader">
        <span :class="$style.sectionIcon">🧠</span>
        <h2 :class="$style.sectionTitle">Installed AI Models</h2>
        <span :class="$style.sectionCount">{{ ollamaModels.length }} models</span>
      </div>

      <div v-if="!ollamaOnline" :class="$style.emptyState">
        <div :class="$style.emptyIcon">🔴</div>
        <div :class="$style.emptyText">Ollama is offline</div>
        <code :class="$style.emptyCode">ollama serve</code>
      </div>
      <div v-else-if="ollamaModels.length === 0" :class="$style.emptyState">
        <div :class="$style.emptyIcon">📦</div>
        <div :class="$style.emptyText">No models installed</div>
        <code :class="$style.emptyCode">ollama pull llama3.1:8b</code>
      </div>

      <div v-else :class="$style.tableWrap">
        <table :class="$style.table">
          <thead>
            <tr>
              <th>Model</th>
              <th>Size</th>
              <th>Parameters</th>
              <th>Quantization</th>
              <th>Status</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="m in ollamaModels" :key="m.name" :class="$style.tableRow">
              <td>
                <div :class="$style.modelCell">
                  <span :class="$style.modelIcon">🤖</span>
                  <span :class="$style.modelName">{{ m.name }}</span>
                </div>
              </td>
              <td>{{ formatBytes(m.size) }}</td>
              <td>{{ m.details?.parameter_size || '—' }}</td>
              <td>
                <span :class="$style.quantTag">{{ m.details?.quantization_level || '—' }}</span>
              </td>
              <td>
                <span :class="m.name === activeModel ? $style.activePill : $style.idlePill">
                  {{ m.name === activeModel ? '● Active' : '○ Idle' }}
                </span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Architecture -->
    <div :class="$style.section">
      <div :class="$style.sectionHeader">
        <span :class="$style.sectionIcon">🏗️</span>
        <h2 :class="$style.sectionTitle">System Architecture</h2>
      </div>
      <div :class="$style.archGrid">
        <div v-for="item in [
          { icon: '🗄️', label: 'Database', value: 'SQLite' },
          { icon: '⚡', label: 'Runtime', value: 'Node.js 22' },
          { icon: '🎨', label: 'Frontend', value: 'Vue 3 + Vite' },
          { icon: '🤖', label: 'AI Engine', value: 'Ollama (local)' },
          { icon: '🔒', label: 'Privacy', value: '100% Offline' },
          { icon: '📦', label: 'Packages', value: 'pnpm' },
        ]" :key="item.label" :class="$style.archCard">
          <div :class="$style.archIcon">{{ item.icon }}</div>
          <div :class="$style.archLabel">{{ item.label }}</div>
          <div :class="$style.archValue">{{ item.value }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" module>
/* === Layout === */
.page {
  min-height: 100vh;
  background: #0e0e16;
  padding: 2rem 2.5rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #e2e8f0;
}

/* === Header === */
.pageHeader {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 2rem;
}

.pageTitle {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.pageTitleIcon {
  font-size: 2.5rem;
  filter: drop-shadow(0 0 12px #f59e0b);
}

.h1 {
  font-size: 1.8rem;
  font-weight: 800;
  margin: 0;
  background: linear-gradient(135deg, #e2e8f0, #94a3b8);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.subtitle {
  margin: 0.2rem 0 0;
  color: #64748b;
  font-size: 0.85rem;
}

.headerRight {
  display: flex;
  align-items: center;
  gap: 0.875rem;
}

.lastUpdated {
  font-size: 0.78rem;
  color: #475569;
}

.refreshBtn {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  background: rgba(99, 102, 241, 0.15);
  border: 1px solid rgba(99, 102, 241, 0.4);
  color: #a5b4fc;
  border-radius: 8px;
  padding: 0.45rem 1rem;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    background: rgba(99, 102, 241, 0.25);
    border-color: rgba(99, 102, 241, 0.6);
  }
}

.refreshIcon {
  font-size: 1rem;
}

.refreshing .refreshIcon {
  display: inline-block;
  animation: spin 0.7s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to   { transform: rotate(360deg); }
}

/* === Cards === */
.cardGrid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1.25rem;
  margin-bottom: 2.5rem;
}

.card {
  position: relative;
  border-radius: 16px;
  padding: 1.4rem 1.4rem 1.2rem;
  border: 1px solid rgba(255, 255, 255, 0.06);
  background: rgba(255, 255, 255, 0.04);
  backdrop-filter: blur(12px);
  overflow: hidden;
  transition: transform 0.2s, box-shadow 0.2s;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 12px 32px rgba(0, 0, 0, 0.35);
  }
}

.cardGlow {
  position: absolute;
  top: -60px;
  right: -60px;
  width: 140px;
  height: 140px;
  border-radius: 50%;
  opacity: 0.25;
  filter: blur(40px);
  pointer-events: none;
}

.cardSuccess .cardGlow { background: #22c55e; }
.cardDanger  .cardGlow { background: #ef4444; }
.cardIndigo  .cardGlow { background: #6366f1; }
.cardPurple  .cardGlow { background: #a855f7; }
.cardTeal    .cardGlow { background: #14b8a6; }

.cardSuccess { border-color: rgba(34, 197, 94, 0.2); }
.cardDanger  { border-color: rgba(239, 68, 68, 0.2); }
.cardIndigo  { border-color: rgba(99, 102, 241, 0.2); }
.cardPurple  { border-color: rgba(168, 85, 247, 0.2); }
.cardTeal    { border-color: rgba(20, 184, 166, 0.2); }

.cardTop {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.875rem;
}

.cardEmoji { font-size: 1.6rem; }

.cardTopRight {
  font-size: 0.82rem;
  color: #64748b;
  font-weight: 600;
}

.cardLabel {
  font-size: 0.7rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #64748b;
  margin-bottom: 0.3rem;
}

.cardValue {
  font-size: 1.45rem;
  font-weight: 700;
  color: #f1f5f9;
  margin-bottom: 0.25rem;
}

.cardValueLarge {
  font-size: 2rem;
}

.cardSub {
  font-size: 0.78rem;
  color: #475569;
  margin-top: 0.35rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* === Badge === */
.badge {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.2rem 0.6rem;
  border-radius: 999px;
}

.badgeGreen {
  background: rgba(34, 197, 94, 0.12);
  color: #4ade80;
  border: 1px solid rgba(34, 197, 94, 0.25);
}

.badgeRed {
  background: rgba(239, 68, 68, 0.12);
  color: #f87171;
  border: 1px solid rgba(239, 68, 68, 0.25);
}

.badgeDot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  display: inline-block;
  animation: pulse 2s infinite;
}

.badgeGreen .badgeDot { background: #4ade80; }
.badgeRed   .badgeDot { background: #f87171; animation: none; }

@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50%       { opacity: 0.5; transform: scale(0.8); }
}

/* === Progress === */
.progressTrack {
  height: 5px;
  background: rgba(255, 255, 255, 0.07);
  border-radius: 999px;
  margin-top: 0.75rem;
  overflow: hidden;
}

.progressFill {
  height: 100%;
  border-radius: 999px;
  background: linear-gradient(90deg, #6366f1, #8b5cf6);
  transition: width 0.6s ease;
}

/* === Section === */
.section { margin-bottom: 2.5rem; }

.sectionHeader {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  margin-bottom: 1rem;
}

.sectionIcon { font-size: 1.2rem; }

.sectionTitle {
  font-size: 1rem;
  font-weight: 700;
  color: #e2e8f0;
  margin: 0;
}

.sectionCount {
  margin-left: auto;
  font-size: 0.78rem;
  color: #475569;
  background: rgba(255,255,255,0.05);
  padding: 0.2rem 0.6rem;
  border-radius: 6px;
}

/* === Empty State === */
.emptyState {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  padding: 2.5rem;
  background: rgba(255,255,255,0.02);
  border: 1px dashed rgba(255,255,255,0.08);
  border-radius: 12px;
  color: #475569;
  text-align: center;
}

.emptyIcon { font-size: 1.75rem; }
.emptyText { font-size: 0.9rem; font-weight: 500; }
.emptyCode {
  background: rgba(255,255,255,0.06);
  padding: 0.25rem 0.6rem;
  border-radius: 6px;
  font-family: 'Consolas', monospace;
  font-size: 0.82rem;
  color: #818cf8;
}

/* === Table === */
.tableWrap {
  border: 1px solid rgba(255,255,255,0.06);
  border-radius: 12px;
  overflow: hidden;
}

.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.88rem;

  th {
    text-align: left;
    padding: 0.8rem 1.1rem;
    background: rgba(255,255,255,0.04);
    color: #64748b;
    font-weight: 600;
    font-size: 0.72rem;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    border-bottom: 1px solid rgba(255,255,255,0.06);
  }

  td {
    padding: 0.875rem 1.1rem;
    color: #cbd5e1;
    border-bottom: 1px solid rgba(255,255,255,0.04);
  }
}

.tableRow:hover td { background: rgba(255,255,255,0.03); }
.tableRow:last-child td { border-bottom: none; }

.modelCell {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}
.modelIcon { font-size: 1rem; }
.modelName { font-weight: 600; color: #e2e8f0; }

.quantTag {
  background: rgba(99,102,241,0.12);
  color: #a5b4fc;
  border: 1px solid rgba(99,102,241,0.2);
  border-radius: 5px;
  padding: 0.15rem 0.5rem;
  font-size: 0.75rem;
  font-weight: 600;
  font-family: monospace;
}

.activePill {
  color: #4ade80;
  font-weight: 700;
  font-size: 0.82rem;
}

.idlePill {
  color: #475569;
  font-size: 0.82rem;
}

/* === Architecture === */
.archGrid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 1rem;
}

.archCard {
  padding: 1.2rem;
  background: rgba(255,255,255,0.03);
  border: 1px solid rgba(255,255,255,0.06);
  border-radius: 12px;
  text-align: center;
  transition: all 0.2s;

  &:hover {
    background: rgba(255,255,255,0.06);
    border-color: rgba(99,102,241,0.3);
    transform: translateY(-2px);
  }
}

.archIcon  { font-size: 1.75rem; margin-bottom: 0.5rem; }
.archLabel { font-size: 0.68rem; color: #64748b; text-transform: uppercase; letter-spacing: 0.06em; }
.archValue { font-size: 0.88rem; font-weight: 700; color: #cbd5e1; margin-top: 0.25rem; }
</style>
