<script setup>
/* global axios */
import { ref, computed, onMounted, watch } from 'vue';
import { useRouter } from 'vue-router';
import { useStore } from 'vuex';
import { useMapGetter } from 'dashboard/composables/store.js';
import { useAccount } from 'dashboard/composables/useAccount';

const { accountId } = useAccount();
const router = useRouter();
const store = useStore();

const labels = useMapGetter('labels/getLabels');

// columns: { [labelTitle]: { conversations, loading, color, title } }
const columns = ref({});
const isRefreshing = ref(false);

// Card drag state
const dragState = ref({ convId: null, fromLabel: null, conv: null });
const dropTarget = ref('');

// Column order & visibility (persisted to localStorage)
const columnOrder = ref([]);
const hiddenColumns = ref(new Set());
const showColumnManager = ref(false);

// Column drag state
const colDragState = ref({ from: null });
const colDropTarget = ref('');

// Card popup / notes
const selectedCard = ref(null);
const cardNote = ref('');
const notes = ref({});
const savingNote = ref(false);
const noteSaved = ref(false); // feedback visual do botão Salvar nota
const loadedContactNote = ref(''); // rastreia nota carregada da API (evita duplicatas)
const cardHistory = ref({ loading: false, timeline: [] });

// Data em que a etiqueta atual (última) do card foi atribuída — id -> epoch segundos.
// Atualizado com precisão no drag-and-drop; no primeiro encontro do card via fetch,
// usa a data de primeiro avistamento como aproximação (não há essa data na API de listagem).
const labelEnteredAt = ref({});

// --- Persistence ---
const storageKey = suffix => `kanban-${suffix}-${accountId.value}`;

const loadPersisted = () => {
  try {
    const order = localStorage.getItem(storageKey('order'));
    if (order) columnOrder.value = JSON.parse(order);
    const hidden = localStorage.getItem(storageKey('hidden'));
    if (hidden) hiddenColumns.value = new Set(JSON.parse(hidden));
    const savedNotes = localStorage.getItem(storageKey('notes'));
    if (savedNotes) notes.value = JSON.parse(savedNotes);
    const savedEnteredAt = localStorage.getItem(storageKey('labelEnteredAt'));
    if (savedEnteredAt) labelEnteredAt.value = JSON.parse(savedEnteredAt);
  } catch { /* ignore */ }
};

const saveOrder = () => {
  try { localStorage.setItem(storageKey('order'), JSON.stringify(columnOrder.value)); } catch { /**/ }
};
const saveHidden = () => {
  try { localStorage.setItem(storageKey('hidden'), JSON.stringify([...hiddenColumns.value])); } catch { /**/ }
};
const saveNotes = () => {
  try { localStorage.setItem(storageKey('notes'), JSON.stringify(notes.value)); } catch { /**/ }
};
const saveLabelEnteredAt = () => {
  try { localStorage.setItem(storageKey('labelEnteredAt'), JSON.stringify(labelEnteredAt.value)); } catch { /**/ }
};

// --- Contact Notes API ---
// Salva nota no contato via API; ignora se o conteúdo não mudou (evita duplicatas)
const saveContactNote = async (contactId, content) => {
  if (!contactId || !content?.trim()) return;
  if (content.trim() === loadedContactNote.value.trim()) return;
  try {
    await axios.post(
      `/api/v1/accounts/${accountId.value}/contacts/${contactId}/notes`,
      { note: { content } }
    );
    loadedContactNote.value = content.trim();
  } catch { /* ignora silenciosamente */ }
};

// Carrega a nota mais recente do contato da API
const loadContactNote = async contactId => {
  if (!contactId) return null;
  try {
    const res = await axios.get(
      `/api/v1/accounts/${accountId.value}/contacts/${contactId}/notes`
    );
    const list = Array.isArray(res.data?.payload) ? res.data.payload
      : Array.isArray(res.data) ? res.data : [];
    if (!list.length) return null;
    const sorted = [...list].sort((a, b) => (b.created_at || 0) - (a.created_at || 0));
    return sorted[0]?.content || null;
  } catch { return null; }
};

// --- Column sync ---
const syncColumns = labelList => {
  const updated = {};
  labelList.forEach(label => {
    updated[label.title] = {
      title: label.title,
      color: label.color || '#6b7280',
      conversations: columns.value[label.title]?.conversations || [],
      loading: false,
    };
  });
  columns.value = updated;

  const titles = labelList.map(l => l.title);
  const kept = columnOrder.value.filter(t => titles.includes(t));
  const added = titles.filter(t => !kept.includes(t));
  columnOrder.value = [...kept, ...added];
  saveOrder();
};

const toggleHidden = title => {
  if (hiddenColumns.value.has(title)) {
    hiddenColumns.value.delete(title);
  } else {
    hiddenColumns.value.add(title);
  }
  hiddenColumns.value = new Set(hiddenColumns.value); // trigger reactivity
  saveHidden();
};

onMounted(() => {
  loadPersisted();
  store.dispatch('labels/get');
});

watch(labels, newLabels => {
  if (newLabels.length > 0) {
    syncColumns(newLabels);
    // Só faz fetch inicial uma vez. Atualizações subsequentes de labels
    // (ex: mudança de cor) não recarregam todas as conversas.
    if (!isRefreshing.value && Object.values(columns.value).every(c => c.conversations.length === 0)) {
      fetchAll();
    }
  }
});

// --- Fetch ---
const fetchColumn = async labelTitle => {
  const col = columns.value[labelTitle];
  if (!col) return;
  col.loading = true;
  try {
    const MAX_PAGES = 5; // ~125 conversas por coluna; aumentar se precisar de mais histórico
    const getPage = page => axios.get(
      `/api/v1/accounts/${accountId.value}/conversations`,
      { params: { labels: labelTitle, status: 'open', assignee_type: 'all', page } }
    );

    const first = await getPage(1);
    let all = first.data?.data?.payload || [];

    // all_count já vem filtrado por label+status (ConversationFinder aplica o
    // filtro antes de contar), então dá para buscar as páginas restantes em
    // paralelo em vez de uma sequência de requisições que espera cada anterior.
    const totalCount = first.data?.data?.meta?.all_count ?? all.length;
    const totalPages = Math.min(Math.ceil(totalCount / 25), MAX_PAGES);
    if (totalPages > 1) {
      const rest = await Promise.all(
        Array.from({ length: totalPages - 1 }, (_, i) => getPage(i + 2))
      );
      rest.forEach(r => { all = all.concat(r.data?.data?.payload || []); });
    }
    col.conversations = all;

    let touched = false;
    all.forEach(c => {
      if (labelEnteredAt.value[c.id] === undefined) {
        labelEnteredAt.value[c.id] = c.timestamp || c.created_at || Math.floor(Date.now() / 1000);
        touched = true;
      }
    });
    if (touched) saveLabelEnteredAt();
  } catch {
    col.conversations = [];
  } finally {
    col.loading = false;
  }
};

const fetchAll = async () => {
  isRefreshing.value = true;
  await Promise.all(Object.keys(columns.value).map(fetchColumn));
  isRefreshing.value = false;
};

// --- Card drag-and-drop ---
const onDragStart = (event, conv, fromLabel) => {
  dragState.value = { convId: conv.id, fromLabel, conv };
  event.dataTransfer.effectAllowed = 'move';
};

const onDrop = async (event, targetLabel) => {
  dropTarget.value = '';
  const { convId, fromLabel, conv } = dragState.value;
  dragState.value = { convId: null, fromLabel: null, conv: null };
  if (!convId || fromLabel === targetLabel) return;

  const currentLabels = Array.isArray(conv.labels) ? [...conv.labels] : [];
  const newLabels = currentLabels.filter(l => l !== fromLabel);
  if (!newLabels.includes(targetLabel)) newLabels.push(targetLabel);

  const fromCol = columns.value[fromLabel];
  const toCol = columns.value[targetLabel];
  if (fromCol) fromCol.conversations = fromCol.conversations.filter(c => c.id !== convId);
  if (toCol) toCol.conversations.unshift({ ...conv, labels: newLabels });

  // Movido manualmente agora: data exata, ao contrário da aproximação do fetch inicial
  labelEnteredAt.value[convId] = Math.floor(Date.now() / 1000);
  saveLabelEnteredAt();

  try {
    await axios.post(
      `/api/v1/accounts/${accountId.value}/conversations/${convId}/labels`,
      { labels: newLabels }
    );
  } catch {
    if (fromCol) fromCol.conversations.unshift(conv);
    if (toCol) toCol.conversations = toCol.conversations.filter(c => c.id !== convId);
  }
};

// --- Column drag-and-drop ---
const onColDragStart = (event, title) => {
  event.stopPropagation();
  colDragState.value = { from: title };
  event.dataTransfer.effectAllowed = 'move';
};

const onColDragEnd = () => {
  colDragState.value = { from: null };
  colDropTarget.value = '';
};

const onColDrop = (event, toTitle) => {
  colDropTarget.value = '';
  const from = colDragState.value.from;
  colDragState.value = { from: null };
  if (!from || from === toTitle) return;

  const order = [...columnOrder.value];
  const fi = order.indexOf(from);
  const ti = order.indexOf(toTitle);
  if (fi === -1 || ti === -1) return;
  order.splice(fi, 1);
  order.splice(ti, 0, from);
  columnOrder.value = order;
  saveOrder();
};

// Unified drag zone handlers on each column container
const onZoneDragOver = (event, title) => {
  event.preventDefault();
  if (colDragState.value.from) {
    if (colDragState.value.from !== title) colDropTarget.value = title;
  } else {
    dropTarget.value = title;
  }
};

const onZoneDragLeave = event => {
  if (!event.currentTarget.contains(event.relatedTarget)) {
    dropTarget.value = '';
    colDropTarget.value = '';
  }
};

const onZoneDrop = (event, title) => {
  event.preventDefault();
  if (colDragState.value.from) {
    onColDrop(event, title);
  } else {
    onDrop(event, title);
  }
};

// --- Card popup ---
const openCard = async conv => {
  selectedCard.value = conv;
  const localNote = notes.value[conv.id] || '';
  cardNote.value = localNote;
  loadedContactNote.value = localNote;
  cardHistory.value = { loading: true, timeline: [] };

  // Busca nota do contato e histórico de etiquetas em paralelo
  const [, msgs] = await Promise.all([
    // Nota do contato
    (async () => {
      if (!localNote) {
        const contactId = conv.meta?.sender?.id;
        if (contactId) {
          const remoteNote = await loadContactNote(contactId);
          if (remoteNote) {
            cardNote.value = remoteNote;
            loadedContactNote.value = remoteNote;
            notes.value[conv.id] = remoteNote;
            saveNotes();
          }
        }
      }
    })(),
    // Histórico de etiquetas
    axios.get(`/api/v1/accounts/${accountId.value}/conversations/${conv.id}/messages`)
      .then(res => Array.isArray(res.data?.payload) ? res.data.payload : [])
      .catch(() => []),
  ]);

  cardHistory.value = { loading: false, timeline: buildLabelTimeline(msgs, labels.value) };
};

const saveNoteExplicit = async () => {
  if (!selectedCard.value) return;
  const contactId = selectedCard.value.meta?.sender?.id;
  const content = cardNote.value.trim();
  if (!content) return;
  savingNote.value = true;
  notes.value[selectedCard.value.id] = cardNote.value;
  saveNotes();
  await saveContactNote(contactId, content);
  savingNote.value = false;
  noteSaved.value = true;
  setTimeout(() => { noteSaved.value = false; }, 2000);
};

const closeCard = async () => {
  if (selectedCard.value) {
    notes.value[selectedCard.value.id] = cardNote.value;
    saveNotes();
    const contactId = selectedCard.value.meta?.sender?.id;
    if (contactId && cardNote.value.trim()) {
      savingNote.value = true;
      await saveContactNote(contactId, cardNote.value.trim());
      savingNote.value = false;
    }
  }
  selectedCard.value = null;
  cardNote.value = '';
  loadedContactNote.value = '';
  noteSaved.value = false;
  cardHistory.value = { loading: false, timeline: [] };
};

const goToConversation = async () => {
  if (!selectedCard.value) return;
  notes.value[selectedCard.value.id] = cardNote.value;
  saveNotes();
  const contactId = selectedCard.value.meta?.sender?.id;
  const id = selectedCard.value.id;
  if (contactId && cardNote.value.trim()) {
    savingNote.value = true;
    await saveContactNote(contactId, cardNote.value.trim());
    savingNote.value = false;
  }
  selectedCard.value = null;
  router.push(`/app/accounts/${accountId.value}/conversations/${id}`);
};

// --- Helpers ---
const relativeTime = ts => {
  if (!ts) return '';
  const diff = Math.floor((Date.now() / 1000 - ts) / 60);
  if (diff < 1) return 'agora';
  if (diff < 60) return `${diff}min`;
  const h = Math.floor(diff / 60);
  if (h < 24) return `${h}h`;
  return `${Math.floor(h / 24)}d`;
};

const labelColumns = computed(() => {
  const ordered = columnOrder.value
    .filter(t => columns.value[t])
    .map(t => columns.value[t]);
  const inOrder = new Set(columnOrder.value);
  Object.values(columns.value).forEach(col => {
    if (!inOrder.has(col.title)) ordered.push(col);
  });
  return ordered;
});

const visibleColumns = computed(() =>
  labelColumns.value.filter(col => !hiddenColumns.value.has(col.title))
);

// --- Filters ---
const showFilters = ref(false);
const filterSearch = ref('');
const filterAssignee = ref('');
const filterDateFrom = ref('');
const filterDateTo = ref('');

const availableAssignees = computed(() => {
  const map = {};
  Object.values(columns.value).forEach(col => {
    col.conversations.forEach(conv => {
      const a = conv.meta?.assignee;
      if (a?.id) map[a.id] = a;
    });
  });
  return Object.values(map).sort((a, b) => a.name.localeCompare(b.name));
});

const hasActiveFilters = computed(
  () =>
    filterSearch.value.trim() !== '' ||
    filterAssignee.value !== '' ||
    filterDateFrom.value !== '' ||
    filterDateTo.value !== ''
);

const clearFilters = () => {
  filterSearch.value = '';
  filterAssignee.value = '';
  filterDateFrom.value = '';
  filterDateTo.value = '';
};

// labelsMap evita O(n) find no template a cada render
const labelsMap = computed(() => {
  const m = {};
  for (const l of labels.value) m[l.title] = l;
  return m;
});

const labelColor = title => labelsMap.value[title]?.color ?? '#6b7280';

// computed evita recalcular filtros 3× por coluna por render
const filteredConvsMap = computed(() => {
  const q = filterSearch.value.trim().toLowerCase();
  const result = {};
  for (const col of labelColumns.value) {
    const deduped = col.conversations.filter(c =>
      Array.isArray(c.labels) && c.labels.length > 0 &&
      c.labels[c.labels.length - 1] === col.title
    );
    let filtered = deduped;
    if (q) {
      filtered = filtered.filter(c =>
        (c.meta?.sender?.name ?? '').toLowerCase().includes(q) ||
        String(c.id).includes(q)
      );
    }
    if (filterAssignee.value) {
      filtered = filtered.filter(c => String(c.meta?.assignee?.id) === String(filterAssignee.value));
    }
    if (filterDateFrom.value) {
      const fromTs = new Date(`${filterDateFrom.value}T00:00:00`).getTime() / 1000;
      filtered = filtered.filter(c => (labelEnteredAt.value[c.id] ?? 0) >= fromTs);
    }
    if (filterDateTo.value) {
      const toTs = new Date(`${filterDateTo.value}T23:59:59`).getTime() / 1000;
      filtered = filtered.filter(c => (labelEnteredAt.value[c.id] ?? 0) <= toTs);
    }
    result[col.title] = { convs: filtered, total: deduped.length };
  }
  return result;
});

const activeFilterCount = computed(() => {
  let n = 0;
  if (filterSearch.value.trim()) n++;
  if (filterAssignee.value) n++;
  if (filterDateFrom.value) n++;
  if (filterDateTo.value) n++;
  return n;
});

// --- Histórico de Etiquetas ---
const historyConvId = ref(null);
const historyData = ref({});

// Interpreta mensagem de atividade e extrai { type: 'added'|'removed', label }
// Formato pt-BR: "{user_name} adicionou {labels}" / "{user_name} removeu {labels}"
// Formato EN:    "{user_name} added {labels}"     / "{user_name} removed {labels}"
const parseActivityLabel = (content, knownTitles) => {
  if (!content) return null;
  const txt = content.trim();

  // Tenta extrair o texto após "adicionou" / "added"
  const addMatch = txt.match(/\badicionou\s+(.+)$/i) || txt.match(/\badded\s+(.+)$/i);
  if (addMatch) {
    const labelStr = addMatch[1].trim();
    // Divide por vírgula (múltiplas etiquetas em um único evento)
    const candidates = labelStr.split(',').map(l => l.trim()).filter(Boolean);
    const filtered = knownTitles
      ? candidates.filter(c => knownTitles.has(c.toLowerCase()))
      : candidates;
    if (filtered.length) return filtered.map(label => ({ type: 'added', label }));
  }

  // Tenta extrair o texto após "removeu" / "removed"
  // Cuidado: "removeu a prioridade" / "removeu a política de SLA X" — filtrar por labels conhecidas
  const remMatch = txt.match(/\bremoveu\s+(.+)$/i) || txt.match(/\bremoved\s+(.+)$/i);
  if (remMatch) {
    const labelStr = remMatch[1].trim();
    const candidates = labelStr.split(',').map(l => l.trim()).filter(Boolean);
    const filtered = knownTitles
      ? candidates.filter(c => knownTitles.has(c.toLowerCase()))
      : candidates;
    if (filtered.length) return filtered.map(label => ({ type: 'removed', label }));
  }

  return null;
};

// Reconstrói timeline de etiquetas a partir das mensagens de atividade (message_type === 2)
const buildLabelTimeline = (msgs, labelList = []) => {
  // Set de títulos conhecidos para filtrar falsos positivos (ex: "a prioridade")
  const knownTitles = labelList.length
    ? new Set(labelList.map(l => l.title.toLowerCase()))
    : null;

  const events = [];
  for (const m of msgs) {
    if (m.message_type !== 2) continue;
    const parsed = parseActivityLabel(m.content, knownTitles);
    if (parsed) {
      for (const ev of parsed) {
        events.push({ ...ev, ts: m.created_at });
      }
    }
  }
  events.sort((a, b) => a.ts - b.ts);

  const active = {}; // label -> startTs
  const timeline = [];
  for (const ev of events) {
    if (ev.type === 'added' && !active[ev.label]) {
      active[ev.label] = ev.ts;
    } else if (ev.type === 'removed' && active[ev.label]) {
      timeline.push({ label: ev.label, startTs: active[ev.label], endTs: ev.ts, active: false });
      delete active[ev.label];
    }
  }
  // Etiquetas ainda ativas (não removidas)
  const nowSec = Math.floor(Date.now() / 1000);
  for (const [label, startTs] of Object.entries(active)) {
    timeline.push({ label, startTs, endTs: null, active: true, nowSec });
  }
  return timeline.sort((a, b) => a.startTs - b.startTs);
};

const fmtDuration = secs => {
  const m = Math.round(Math.abs(secs) / 60);
  if (m < 60) return `${m}min`;
  const h = Math.floor(m / 60);
  const rm = m % 60;
  if (h < 24) return rm > 0 ? `${h}h ${rm}min` : `${h}h`;
  const d = Math.floor(h / 24);
  const rh = h % 24;
  return rh > 0 ? `${d}d ${rh}h` : `${d}d`;
};

const fmtDateTime = ts => {
  if (!ts) return '';
  return new Date(ts * 1000).toLocaleString('pt-BR', {
    day: '2-digit', month: '2-digit', hour: '2-digit', minute: '2-digit',
  });
};

const openHistory = async (conv, event) => {
  event.stopPropagation();
  historyConvId.value = conv.id;
  // Invalida cache se a conversa foi movida desde a última busca (remove cache)
  // Re-fetch sempre para garantir dados atualizados após drag-and-drop
  historyData.value = {
    ...historyData.value,
    [conv.id]: { loading: true, timeline: [], convName: conv.meta?.sender?.name },
  };
  try {
    const res = await axios.get(
      `/api/v1/accounts/${accountId.value}/conversations/${conv.id}/messages`
    );
    const msgs = Array.isArray(res.data?.payload) ? res.data.payload : [];
    historyData.value = {
      ...historyData.value,
      [conv.id]: { loading: false, timeline: buildLabelTimeline(msgs, labels.value), convName: conv.meta?.sender?.name },
    };
  } catch {
    historyData.value = {
      ...historyData.value,
      [conv.id]: { loading: false, timeline: [], convName: conv.meta?.sender?.name },
    };
  }
};

const closeHistory = () => { historyConvId.value = null; };
const currentHistory = computed(() => historyData.value[historyConvId.value] || null);
</script>

<template>
  <section
    class="flex flex-col w-full h-full bg-n-surface-1 overflow-hidden"
    @click="showColumnManager = false; showFilters = false"
  >
    <!-- Header -->
    <div class="flex items-center justify-between px-6 py-4 border-b border-n-weak flex-shrink-0">
      <div class="flex items-center gap-3">
        <span class="i-lucide-kanban size-5 text-n-slate-11" />
        <h1 class="text-base font-semibold text-n-slate-12">Kanban de Etiquetas</h1>
      </div>
      <div class="flex items-center gap-2">
        <!-- Column manager -->
        <div class="relative" @click.stop>
          <button
            class="flex items-center gap-1.5 px-3 py-1.5 text-sm rounded-lg border border-n-weak bg-n-solid-2 hover:bg-n-solid-3 text-n-slate-11 transition-colors"
            @click="showColumnManager = !showColumnManager"
          >
            <span class="i-lucide-columns-3 size-3.5" />
            Colunas
          </button>
          <div
            v-if="showColumnManager"
            class="absolute right-0 top-full mt-1 z-20 min-w-[200px] bg-n-solid-1 border border-n-weak rounded-xl shadow-lg py-2"
          >
            <div class="px-3 py-1 text-xs text-n-slate-9 font-medium uppercase tracking-wide mb-1">
              Exibir / Ocultar
            </div>
            <label
              v-for="col in labelColumns"
              :key="col.title"
              class="flex items-center gap-2.5 px-3 py-2 hover:bg-n-solid-2 cursor-pointer select-none"
            >
              <input
                type="checkbox"
                :checked="!hiddenColumns.has(col.title)"
                @change="toggleHidden(col.title)"
              />
              <span class="size-2.5 rounded-sm flex-shrink-0" :style="{ backgroundColor: col.color }" />
              <span class="text-sm text-n-slate-12 truncate">{{ col.title }}</span>
            </label>
          </div>
        </div>
        <!-- Filters toggle -->
        <button
          class="flex items-center gap-1.5 px-3 py-1.5 text-sm rounded-lg border transition-colors relative"
          :class="
            hasActiveFilters
              ? 'border-[var(--color-woot-500)] bg-[var(--color-woot-500)]/10 text-[var(--color-woot-600)]'
              : 'border-n-weak bg-n-solid-2 hover:bg-n-solid-3 text-n-slate-11'
          "
          @click.stop="showFilters = !showFilters"
        >
          <span class="i-lucide-sliders-horizontal size-3.5" />
          Filtros
          <span
            v-if="activeFilterCount > 0"
            class="absolute -top-1.5 -right-1.5 size-4 rounded-full bg-[var(--color-woot-500)] text-white text-[10px] font-bold flex items-center justify-center"
          >{{ activeFilterCount }}</span>
        </button>
        <!-- Refresh -->
        <button
          class="flex items-center gap-1.5 px-3 py-1.5 text-sm rounded-lg border border-n-weak bg-n-solid-2 hover:bg-n-solid-3 text-n-slate-11 disabled:opacity-50 transition-colors"
          :disabled="isRefreshing"
          @click="fetchAll"
        >
          <span class="i-lucide-refresh-cw size-3.5" :class="isRefreshing ? 'animate-spin' : ''" />
          Atualizar
        </button>
      </div>
    </div>

    <!-- Filter panel -->
    <div
      v-if="showFilters"
      class="flex items-center gap-3 px-6 py-3 border-b border-n-weak bg-n-solid-1 flex-shrink-0 flex-wrap"
      @click.stop
    >
      <!-- Search -->
      <div class="relative min-w-[220px] flex-1">
        <span class="absolute left-2.5 top-1/2 -translate-y-1/2 i-lucide-search size-3.5 text-n-slate-9 pointer-events-none" />
        <input
          v-model="filterSearch"
          class="w-full text-sm pl-8 pr-3 py-1.5 rounded-lg border border-n-weak bg-n-solid-2 text-n-slate-12 placeholder-n-slate-9 focus:outline-none focus:ring-1 focus:ring-[var(--color-woot-500)] focus:border-[var(--color-woot-500)]"
          placeholder="Buscar por contato ou #ID..."
        />
        <button
          v-if="filterSearch"
          class="absolute right-2 top-1/2 -translate-y-1/2 text-n-slate-8 hover:text-n-slate-11"
          @click="filterSearch = ''"
        >
          <span class="i-lucide-x size-3.5" />
        </button>
      </div>

      <!-- Assignee filter -->
      <div class="relative min-w-[180px]">
        <span class="absolute left-2.5 top-1/2 -translate-y-1/2 i-lucide-circle-user size-3.5 text-n-slate-9 pointer-events-none" />
        <select
          v-model="filterAssignee"
          class="w-full text-sm pl-8 pr-3 py-1.5 rounded-lg border border-n-weak bg-n-solid-2 text-n-slate-12 focus:outline-none focus:ring-1 focus:ring-[var(--color-woot-500)] focus:border-[var(--color-woot-500)] appearance-none cursor-pointer"
        >
          <option value="">Todos os responsáveis</option>
          <option v-for="a in availableAssignees" :key="a.id" :value="a.id">{{ a.name }}</option>
        </select>
        <span class="absolute right-2.5 top-1/2 -translate-y-1/2 i-lucide-chevron-down size-3.5 text-n-slate-9 pointer-events-none" />
      </div>

      <!-- Date filter (data em que a etiqueta atual foi atribuída) -->
      <div class="flex items-center gap-1.5">
        <span class="i-lucide-calendar size-3.5 text-n-slate-9 flex-shrink-0" />
        <input
          v-model="filterDateFrom"
          type="date"
          title="Etiqueta atribuída a partir de"
          class="text-sm px-2 py-1.5 rounded-lg border border-n-weak bg-n-solid-2 text-n-slate-12 focus:outline-none focus:ring-1 focus:ring-[var(--color-woot-500)] focus:border-[var(--color-woot-500)]"
        />
        <span class="text-xs text-n-slate-9">até</span>
        <input
          v-model="filterDateTo"
          type="date"
          title="Etiqueta atribuída até"
          class="text-sm px-2 py-1.5 rounded-lg border border-n-weak bg-n-solid-2 text-n-slate-12 focus:outline-none focus:ring-1 focus:ring-[var(--color-woot-500)] focus:border-[var(--color-woot-500)]"
        />
      </div>

      <!-- Clear filters -->
      <button
        v-if="hasActiveFilters"
        class="flex items-center gap-1.5 px-3 py-1.5 text-sm rounded-lg border border-red-200 dark:border-red-800 text-red-600 dark:text-red-400 bg-red-50 dark:bg-red-950 hover:bg-red-100 dark:hover:bg-red-900 transition-colors flex-shrink-0"
        @click="clearFilters"
      >
        <span class="i-lucide-x size-3.5" />
        Limpar filtros
      </button>

      <!-- Results summary -->
      <span v-if="hasActiveFilters" class="text-xs text-n-slate-9 ml-auto flex-shrink-0">
        {{ visibleColumns.reduce((t, col) => t + filteredConvsMap[col.title].convs.length, 0) }} conversa(s) encontrada(s)
      </span>
    </div>

    <!-- Empty state -->
    <div
      v-if="labelColumns.length === 0"
      class="flex flex-col items-center justify-center flex-1 gap-3 text-n-slate-10"
    >
      <span class="i-lucide-tag size-10 opacity-30" />
      <p class="text-sm">Nenhuma etiqueta cadastrada.</p>
      <p class="text-xs text-n-slate-9">Crie etiquetas em Configurações → Etiquetas.</p>
    </div>

    <!-- Kanban board -->
    <div
      v-else
      class="flex flex-row gap-3 p-4 overflow-x-auto flex-1 min-h-0 items-start"
    >
      <div
        v-for="col in visibleColumns"
        :key="col.title"
        class="flex flex-col rounded-xl border min-w-[272px] max-w-[272px] h-full transition-all bg-n-solid-2"
        :class="[
          dropTarget === col.title
            ? 'border-[var(--color-woot-500)] ring-2 ring-[var(--color-woot-500)] ring-opacity-30'
            : colDropTarget === col.title
              ? 'border-blue-400 ring-2 ring-blue-400 ring-opacity-30'
              : 'border-n-weak',
          colDragState.from === col.title ? 'opacity-40' : '',
        ]"
        @dragover="onZoneDragOver($event, col.title)"
        @dragleave="onZoneDragLeave"
        @drop="onZoneDrop($event, col.title)"
      >
        <!-- Column header with drag handle -->
        <div class="flex items-center justify-between px-3 py-3 border-b border-n-weak flex-shrink-0">
          <div
            class="flex items-center gap-2 min-w-0 flex-1 cursor-grab active:cursor-grabbing"
            draggable="true"
            @dragstart="onColDragStart($event, col.title)"
            @dragend="onColDragEnd"
          >
            <span class="i-lucide-grip-vertical size-3.5 text-n-slate-8 flex-shrink-0" />
            <span class="size-2.5 rounded-sm flex-shrink-0" :style="{ backgroundColor: col.color }" />
            <span class="text-sm font-medium text-n-slate-12 truncate" :title="col.title">
              {{ col.title }}
            </span>
          </div>
          <div class="flex items-center gap-1 flex-shrink-0 ml-2">
            <span
              class="min-w-[20px] text-center px-1.5 py-0.5 rounded-full text-xs font-medium transition-colors"
              :class="
                hasActiveFilters && !col.loading
                  ? 'bg-[var(--color-woot-500)]/15 text-[var(--color-woot-600)]'
                  : 'bg-n-solid-3 text-n-slate-10'
              "
            >
              {{ col.loading ? '…' : (hasActiveFilters ? filteredConvsMap[col.title].convs.length + '/' + filteredConvsMap[col.title].total : filteredConvsMap[col.title].total) }}
            </span>
            <button
              class="p-1 rounded text-n-slate-8 hover:text-n-slate-11 hover:bg-n-solid-3 transition-colors"
              title="Ocultar coluna"
              @click.stop="toggleHidden(col.title)"
            >
              <span class="i-lucide-eye-off size-3.5" />
            </button>
          </div>
        </div>

        <!-- Cards area -->
        <div class="flex flex-col gap-2 p-2 overflow-y-auto flex-1 min-h-0">
          <div v-if="col.loading" class="flex items-center justify-center py-10">
            <span class="i-lucide-loader-circle size-5 text-n-slate-9 animate-spin" />
          </div>
          <div
            v-else-if="filteredConvsMap[col.title].convs.length === 0"
            class="flex flex-col items-center justify-center py-10 gap-2 text-n-slate-9"
          >
            <span class="i-lucide-inbox size-6 opacity-30" />
            <p class="text-xs">{{ hasActiveFilters ? 'Nenhum resultado' : 'Sem conversas' }}</p>
            <button
              v-if="hasActiveFilters"
              class="text-xs text-[var(--color-woot-500)] underline hover:no-underline"
              @click.stop="clearFilters"
            >Limpar filtros</button>
          </div>
          <template v-else>
            <div
              v-for="conv in filteredConvsMap[col.title].convs"
              :key="conv.id"
              draggable="true"
              class="rounded-lg bg-n-solid-1 border border-n-weak p-3 cursor-grab active:cursor-grabbing hover:border-n-strong hover:shadow-sm transition-all select-none"
              @dragstart="onDragStart($event, conv, col.title)"
              @click.stop="openCard(conv)"
            >
              <!-- Contact + time -->
              <div class="flex items-start justify-between gap-2 mb-1.5">
                <div class="flex items-center gap-1.5 min-w-0">
                  <span class="text-sm font-medium text-n-slate-12 leading-tight truncate">
                    {{ conv.meta?.sender?.name || 'Contato' }}
                  </span>
                  <span
                    v-if="conv.unread_count > 0"
                    class="flex-shrink-0 min-w-[18px] h-[18px] px-1 rounded-full bg-red-500 text-white text-[10px] font-bold flex items-center justify-center leading-none"
                  >{{ conv.unread_count > 99 ? '99+' : conv.unread_count }}</span>
                </div>
                <div class="flex items-center gap-1 flex-shrink-0 mt-0.5">
                  <button
                    class="p-0.5 rounded text-n-slate-7 hover:text-[var(--color-woot-500)] transition-colors"
                    title="Histórico de etiquetas"
                    @click.stop="openHistory(conv, $event)"
                  >
                    <span class="i-lucide-history size-3" />
                  </button>
                  <span class="text-xs text-n-slate-9">{{ relativeTime(conv.timestamp || conv.created_at) }}</span>
                </div>
              </div>
              <p class="text-xs text-n-slate-9 mb-1.5">#{{ conv.id }}</p>

              <!-- Note preview -->
              <div v-if="notes[conv.id]" class="flex items-start gap-1 mb-2">
                <span class="i-lucide-sticky-note size-3 text-n-slate-8 mt-0.5 flex-shrink-0" />
                <span class="text-[10px] text-n-slate-8 leading-tight line-clamp-2">{{ notes[conv.id] }}</span>
              </div>

              <!-- Label chips -->
              <div v-if="conv.labels && conv.labels.length > 0" class="flex flex-wrap gap-1 mb-2">
                <span
                  v-for="lbl in conv.labels"
                  :key="lbl"
                  class="px-1.5 py-0.5 rounded text-[10px] font-medium"
                  :style="{
                    backgroundColor: labelColor(lbl) + '22',
                    color: labelColor(lbl),
                  }"
                >{{ lbl }}</span>
              </div>

              <!-- Assignee -->
              <div v-if="conv.meta?.assignee" class="flex items-center gap-1.5 mt-1">
                <span class="i-lucide-circle-user size-3 text-n-slate-9" />
                <span class="text-xs text-n-slate-9 truncate">{{ conv.meta.assignee.name }}</span>
              </div>
            </div>
          </template>
        </div>
      </div>
    </div>

    <!-- Card detail modal -->
    <Teleport to="body">
      <div
        v-if="selectedCard"
        class="fixed inset-0 z-50 flex items-center justify-center bg-black/40 backdrop-blur-sm"
        @click.self="closeCard"
      >
        <div class="bg-n-solid-1 rounded-2xl border border-n-weak shadow-2xl w-full max-w-md mx-4 flex flex-col overflow-hidden">
          <!-- Modal header -->
          <div class="flex items-center justify-between px-5 py-4 border-b border-n-weak">
            <div class="flex items-center gap-2 min-w-0">
              <span class="text-sm font-semibold text-n-slate-12 truncate">
                {{ selectedCard.meta?.sender?.name || 'Contato' }}
              </span>
              <span class="text-xs text-n-slate-9 flex-shrink-0">#{{ selectedCard.id }}</span>
            </div>
            <button
              class="p-1 rounded-lg text-n-slate-9 hover:text-n-slate-12 hover:bg-n-solid-3 transition-colors"
              @click="closeCard"
            >
              <span class="i-lucide-x size-4" />
            </button>
          </div>

          <!-- Labels + assignee -->
          <div class="px-5 pt-4 flex flex-wrap items-center gap-2">
            <template v-if="selectedCard.labels?.length">
              <span
                v-for="lbl in selectedCard.labels"
                :key="lbl"
                class="px-2 py-0.5 rounded text-[11px] font-medium"
                :style="{
                  backgroundColor: labelColor(lbl) + '22',
                  color: labelColor(lbl),
                }"
              >{{ lbl }}</span>
            </template>
            <div v-if="selectedCard.meta?.assignee" class="flex items-center gap-1.5 text-n-slate-9">
              <span class="i-lucide-circle-user size-3.5" />
              <span class="text-xs">{{ selectedCard.meta.assignee.name }}</span>
            </div>
          </div>

          <!-- Histórico de etiquetas -->
          <div class="px-5 pt-4">
            <div class="flex items-center gap-1.5 mb-2">
              <span class="i-lucide-history size-3.5 text-n-slate-9" />
              <span class="text-xs font-medium text-n-slate-10">Histórico de etiquetas</span>
            </div>

            <!-- Loading -->
            <div v-if="cardHistory.loading" class="flex items-center gap-2 py-2 text-n-slate-9">
              <span class="i-lucide-loader-circle size-3.5 animate-spin" />
              <span class="text-xs">Carregando...</span>
            </div>

            <!-- Sem histórico -->
            <p v-else-if="!cardHistory.timeline.length" class="text-xs text-n-slate-8 italic">
              Nenhuma movimentação registrada ainda.
            </p>

            <!-- Timeline compacta -->
            <div v-else class="space-y-1.5">
              <div
                v-for="(item, idx) in cardHistory.timeline"
                :key="idx"
                class="flex items-center gap-2 rounded-lg px-2.5 py-2"
                :class="item.active ? 'bg-[var(--color-woot-500)]/8 border border-[var(--color-woot-500)]/20' : 'bg-n-solid-2 border border-n-weak'"
              >
                <!-- Dot colorido -->
                <div
                  class="size-2.5 rounded-full flex-shrink-0"
                  :style="{ backgroundColor: labelColor(item.label) }"
                />
                <!-- Label -->
                <span
                  class="text-[11px] font-medium flex-1 truncate"
                  :style="{ color: labelColor(item.label) }"
                >{{ item.label }}</span>
                <!-- Duração -->
                <span
                  class="text-xs font-semibold flex-shrink-0"
                  :class="item.active ? 'text-[var(--color-woot-500)]' : 'text-n-slate-11'"
                >
                  {{ fmtDuration(item.endTs ? item.endTs - item.startTs : Math.floor(Date.now() / 1000) - item.startTs) }}
                </span>
                <!-- Badge ativo ou data de saída -->
                <span
                  v-if="item.active"
                  class="text-[10px] text-[var(--color-woot-500)] bg-[var(--color-woot-500)]/15 px-1.5 py-0.5 rounded-full font-medium flex-shrink-0"
                >ativo</span>
                <span v-else class="text-[10px] text-n-slate-8 flex-shrink-0 whitespace-nowrap">
                  até {{ fmtDateTime(item.endTs) }}
                </span>
              </div>
            </div>
          </div>

          <!-- Notes textarea -->
          <div class="px-5 pt-4 pb-2">
            <label class="block text-xs font-medium text-n-slate-10 mb-2">
              <span class="i-lucide-sticky-note size-3.5 inline-block mr-1 align-middle" />
              Observações do atendimento
            </label>
            <textarea
              v-model="cardNote"
              rows="5"
              placeholder="Escreva suas observações sobre o andamento deste atendimento..."
              class="w-full rounded-lg border border-n-weak bg-n-solid-2 text-sm text-n-slate-12 placeholder-n-slate-9 px-3 py-2.5 resize-none focus:outline-none focus:border-[var(--color-woot-500)] focus:ring-1 focus:ring-[var(--color-woot-500)] transition-colors"
            />
          </div>

          <!-- Actions -->
          <div class="flex items-center justify-between gap-2 px-5 py-4 border-t border-n-weak">
            <!-- Salvar nota -->
            <button
              class="px-3 py-2 rounded-lg text-sm border transition-colors disabled:opacity-50 flex items-center gap-1.5"
              :class="noteSaved
                ? 'border-green-500/40 bg-green-500/10 text-green-600 dark:text-green-400'
                : 'border-n-weak bg-n-solid-2 hover:bg-n-solid-3 text-n-slate-11'"
              :disabled="savingNote || !cardNote.trim()"
              @click="saveNoteExplicit"
            >
              <span v-if="savingNote" class="i-lucide-loader-circle size-3.5 animate-spin" />
              <span v-else-if="noteSaved" class="i-lucide-check size-3.5" />
              <span v-else class="i-lucide-save size-3.5" />
              {{ noteSaved ? 'Salvo!' : 'Salvar nota' }}
            </button>
            <div class="flex items-center gap-2">
              <button
                class="px-4 py-2 rounded-lg text-sm text-n-slate-11 border border-n-weak bg-n-solid-2 hover:bg-n-solid-3 transition-colors disabled:opacity-50"
                :disabled="savingNote"
                @click="closeCard"
              >
                Fechar
              </button>
              <button
                class="px-4 py-2 rounded-lg text-sm font-medium text-n-white bg-[var(--color-woot-500)] hover:bg-[var(--color-woot-600)] transition-colors flex items-center gap-1.5 shadow-sm disabled:opacity-60"
                :disabled="savingNote"
                @click="goToConversation"
              >
                <span v-if="savingNote" class="i-lucide-loader-circle size-3.5 animate-spin" />
                <span v-else class="i-lucide-message-circle size-3.5" />
                Ir para conversa
              </button>
            </div>
          </div>
        </div>
      </div>
    </Teleport>

    <!-- History Modal — linha do tempo de movimentações entre etiquetas -->
    <Teleport to="body">
      <div
        v-if="historyConvId"
        class="fixed inset-0 z-50 flex items-center justify-center bg-black/40 backdrop-blur-sm"
        @click.self="closeHistory"
      >
        <div class="bg-n-solid-1 rounded-2xl border border-n-weak shadow-2xl w-full max-w-md mx-4 flex flex-col overflow-hidden max-h-[80vh]">
          <!-- Header -->
          <div class="flex items-center justify-between px-5 py-4 border-b border-n-weak flex-shrink-0">
            <div class="flex items-center gap-2 min-w-0">
              <span class="i-lucide-history size-4 text-[var(--color-woot-500)]" />
              <span class="text-sm font-semibold text-n-slate-12">Histórico de etiquetas</span>
              <span v-if="currentHistory?.convName" class="text-xs text-n-slate-9 truncate">
                — {{ currentHistory.convName }}
              </span>
            </div>
            <button
              class="p-1 rounded-lg text-n-slate-9 hover:text-n-slate-12 hover:bg-n-solid-3 transition-colors flex-shrink-0"
              @click="closeHistory"
            >
              <span class="i-lucide-x size-4" />
            </button>
          </div>

          <!-- Body -->
          <div class="overflow-y-auto flex-1 px-5 py-4">
            <!-- Loading -->
            <div v-if="currentHistory?.loading" class="flex items-center justify-center py-10">
              <span class="i-lucide-loader-circle size-5 text-n-slate-9 animate-spin" />
            </div>

            <!-- Sem histórico -->
            <div
              v-else-if="!currentHistory?.timeline?.length"
              class="flex flex-col items-center justify-center py-8 gap-2 text-n-slate-9"
            >
              <span class="i-lucide-clock size-8 opacity-30" />
              <p class="text-sm">Nenhuma movimentação registrada.</p>
              <p class="text-xs text-n-slate-8 text-center">
                O histórico é salvo automaticamente conforme o card é arrastado entre colunas.
              </p>
            </div>

            <!-- Timeline -->
            <div v-else class="space-y-0">
              <div
                v-for="(item, idx) in currentHistory.timeline"
                :key="idx"
                class="flex gap-3"
              >
                <!-- Dot + linha vertical -->
                <div class="flex flex-col items-center flex-shrink-0">
                  <div
                    class="size-3 rounded-full mt-1 ring-2 ring-n-solid-1"
                    :style="{ backgroundColor: labelColor(item.label) }"
                  />
                  <div
                    v-if="idx < currentHistory.timeline.length - 1"
                    class="w-px flex-1 bg-n-weak mt-1"
                    style="min-height: 32px"
                  />
                </div>

                <!-- Conteúdo -->
                <div class="pb-5 min-w-0 flex-1">
                  <div class="flex items-center gap-2 flex-wrap">
                    <!-- Label chip -->
                    <span
                      class="px-2 py-0.5 rounded text-[11px] font-medium"
                      :style="{
                        backgroundColor: labelColor(item.label) + '22',
                        color: labelColor(item.label),
                      }"
                    >{{ item.label }}</span>
                    <!-- Duração -->
                    <span
                      class="text-xs font-semibold"
                      :class="item.active ? 'text-[var(--color-woot-500)]' : 'text-n-slate-11'"
                    >
                      {{ fmtDuration(item.endTs ? item.endTs - item.startTs : Math.floor(Date.now() / 1000) - item.startTs) }}
                    </span>
                    <!-- Badge ativo -->
                    <span
                      v-if="item.active"
                      class="text-[10px] text-[var(--color-woot-500)] bg-[var(--color-woot-500)]/10 px-1.5 py-0.5 rounded-full font-medium"
                    >ativo</span>
                  </div>
                  <!-- Datas de entrada / saída -->
                  <p class="text-[10px] text-n-slate-8 mt-0.5">
                    {{ fmtDateTime(item.startTs) }}
                    <span v-if="item.endTs"> → {{ fmtDateTime(item.endTs) }}</span>
                    <span v-else class="text-[var(--color-woot-500)]"> → agora</span>
                  </p>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Teleport>
  </section>
</template>