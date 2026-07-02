<script setup lang="ts">
import { ref, computed } from 'vue';
import { useSeedrStore } from '../stores/seedr';
import { formatBytes, formatSpeed } from '../utils/format';
import { getTorrentStatusBadge } from '../utils/torrent-status';

const store = useSeedrStore();

type SortField =
  | 'name'
  | 'added'
  | 'size'
  | 'uploadRate'
  | 'uploaded'
  | 'reportedUploaded'
  | 'seeders'
  | 'leechers'
  | 'active'
  | 'completed';

type SortDir = 'desc' | 'asc';

const allowedSortFields: SortField[] = [
  'name',
  'added',
  'size',
  'uploadRate',
  'uploaded',
  'reportedUploaded',
  'seeders',
  'leechers',
  'active',
  'completed'
];

const savedFieldRaw = localStorage.getItem('sortField');
const savedDirRaw = localStorage.getItem('sortDir');

const sortField = ref<SortField>(
  allowedSortFields.includes(savedFieldRaw as SortField)
    ? (savedFieldRaw as SortField)
    : 'name'
);

const sortDir = ref<SortDir>(
  savedDirRaw === 'desc' ? 'desc' : 'asc'
);

if (!savedFieldRaw) localStorage.setItem('sortField', sortField.value);
if (!savedDirRaw) localStorage.setItem('sortDir', sortDir.value);

const showFileName = computed(() => store.config?.showFileName ?? true);

const collapsedGroups = ref(new Set<string>());
const search = ref('');
const confirmingRemove = ref<string | null>(null);
const pendingActions = ref(new Map<string, string>());

let confirmTimer: ReturnType<typeof setTimeout> | undefined;

/* ---------------- SORT ---------------- */

function toggleSort(field: SortField) {
  if (sortField.value === field) {
    sortDir.value = sortDir.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortField.value = field;
    sortDir.value = 'asc';
  }

  localStorage.setItem('sortField', sortField.value);
  localStorage.setItem('sortDir', sortDir.value);
}

function sortIndicator(field: SortField): string {
  if (sortField.value !== field) return '';
  return sortDir.value === 'asc' ? ' ▲' : ' ▼';
}

/* ---------------- GROUP ---------------- */

function toggleCollapse(tracker: string) {
  if (collapsedGroups.value.has(tracker)) {
    collapsedGroups.value.delete(tracker);
  } else {
    collapsedGroups.value.add(tracker);
  }
}

function trackerHost(url: string): string {
  try { return new URL(url).hostname; } catch { return url || 'Unknown'; }
}

function trackerName(hostname: string): string {
  const stripped = hostname.replace(/^(tracker[0-9]*|announce|tr|www)\./, '');
  const parts = stripped.split('.');
  const name = parts.length >= 2 ? parts[parts.length - 2]! : parts[0]!;
  return name.charAt(0).toUpperCase() + name.slice(1);
}

/* ---------------- LIST ---------------- */

const sortedTorrents = computed(() => {
  const q = search.value.toLowerCase().trim();

  let list = q
    ? store.torrents.filter(t =>
        t.name.toLowerCase().includes(q) ||
        t.fileName.toLowerCase().includes(q)
      )
    : [...store.torrents];

  const dir = sortDir.value === 'asc' ? 1 : -1;

  const getValue = (t: any) => {
    switch (sortField.value) {
      case 'name':
        return t.name.toLowerCase();

      case 'added':
        return t.addedIndex;

      case 'size':
        return t.size;

      case 'uploadRate':
        return t.uploadRate || 0;

      case 'uploaded':
        return t.uploaded;

      case 'reportedUploaded':
        return t.reportedUploaded;

      case 'seeders':
        return t.seeders;

      case 'leechers':
        return t.leechers;

      case 'active':
        return t.active ? 1 : 0;

      case 'completed':
        return t.completed ? 1 : 0;

      default:
        return t.name.toLowerCase();
    }
  };

  list.sort((a, b) => {
    const va = getValue(a);
    const vb = getValue(b);

    if (typeof va === 'string') {
      return dir * va.localeCompare(vb);
    }

    return dir * (va - vb);
  });

  return list;
});

const groupedTorrents = computed(() => {
  const groups = new Map<string, typeof sortedTorrents.value>();

  for (const t of sortedTorrents.value) {
    const host = trackerHost(t.tracker);
    if (!groups.has(host)) groups.set(host, []);
    groups.get(host)!.push(t);
  }

  return [...groups.entries()]
    .map(([host, torrents]) => ({
      host,
      name: trackerName(host),
      torrents
    }))
    .sort((a, b) =>
      a.name.localeCompare(b.name, undefined, { sensitivity: 'base' })
    );
});

/* ---------------- STATUS ---------------- */

function torrentStatus(torrent: any) {
  return getTorrentStatusBadge(
    torrent,
    store.status?.running,
    store.isTorrentEligible(torrent)
  );
}

/* ---------------- ACTIONS ---------------- */

function remove(infoHash: string) {
  if (confirmingRemove.value === infoHash) {
    clearTimeout(confirmTimer);
    confirmingRemove.value = null;
    store.removeTorrent(infoHash);
  } else {
    confirmingRemove.value = infoHash;
    clearTimeout(confirmTimer);
    confirmTimer = setTimeout(() => {
      confirmingRemove.value = null;
    }, 3000);
  }
}

async function announce(infoHash: string) {
  await store.forceAnnounce(infoHash);
}

async function pauseTorrent(infoHash: string) {
  pendingActions.value.set(infoHash, 'pause');
  try {
    await store.pauseTorrent(infoHash);
  } finally {
    pendingActions.value.delete(infoHash);
  }
}

async function resumeTorrent(infoHash: string) {
  pendingActions.value.set(infoHash, 'resume');
  try {
    await store.resumeTorrent(infoHash);
  } finally {
    pendingActions.value.delete(infoHash);
  }
}

async function markCompleted(infoHash: string) {
  pendingActions.value.set(infoHash, 'mark');
  try {
    await store.markTorrentCompleted(infoHash);
  } finally {
    pendingActions.value.delete(infoHash);
  }
}

async function unmarkCompleted(infoHash: string) {
  pendingActions.value.set(infoHash, 'unmark');
  try {
    await store.unmarkTorrentCompleted(infoHash);
  } finally {
    pendingActions.value.delete(infoHash);
  }
}
</script>

<template>
  <div class="bg-gray-900 rounded-xl border border-gray-800">

	<!-- HEADER -->
	<div class="px-4 py-3 border-b border-gray-800 flex items-center justify-between gap-3">

	  <h2 class="text-sm font-semibold text-gray-300">Torrents</h2>

      <span class="text-[10px] text-gray-600 ml-2">
        {{ sortField }} {{ sortDir }}
      </span>

	  <div class="flex items-center gap-2">
	
		<!-- SEARCH -->
		<input
		  v-if="store.torrents.length >= 5"
		  v-model="search"
		  type="text"
		  placeholder="Search..."
		  class="bg-gray-800 border border-gray-700 rounded-lg px-2.5 py-1 text-xs text-white placeholder-gray-600 focus:outline-none w-24 sm:w-36"
		/>

		<!-- SORT MENU -->
		<div v-if="store.torrents.length > 1" class="flex items-center gap-1 text-xs text-gray-500">

		  <button
			@click="toggleSort('name')"
			class="px-1.5 py-0.5 rounded"
			:class="sortField === 'name' ? 'text-gray-300 bg-gray-800' : 'hover:text-gray-400'"
		  >
			Name{{ sortIndicator('name') }}
		  </button>

		  <button
			@click="toggleSort('added')"
			class="px-1.5 py-0.5 rounded"
			:class="sortField === 'added' ? 'text-gray-300 bg-gray-800' : 'hover:text-gray-400'"
		  >
			Added{{ sortIndicator('added') }}
		  </button>

		  <button
			@click="toggleSort('uploadRate')"
			class="px-1.5 py-0.5 rounded"
			:class="sortField === 'uploadRate' ? 'text-gray-300 bg-gray-800' : 'hover:text-gray-400'"
		  >
			Speed{{ sortIndicator('uploadRate') }}
		  </button>

		  <button
			@click="toggleSort('uploaded')"
			class="px-1.5 py-0.5 rounded"
			:class="sortField === 'uploaded' ? 'text-gray-300 bg-gray-800' : 'hover:text-gray-400'"
		  >
			Uploaded{{ sortIndicator('uploaded') }}
		  </button>

		  <button
			@click="toggleSort('seeders')"
			class="px-1.5 py-0.5 rounded"
			:class="sortField === 'seeders' ? 'text-gray-300 bg-gray-800' : 'hover:text-gray-400'"
		  >
			Seeds{{ sortIndicator('seeders') }}
		  </button>

		  <button
			@click="toggleSort('leechers')"
			class="px-1.5 py-0.5 rounded"
			:class="sortField === 'leechers' ? 'text-gray-300 bg-gray-800' : 'hover:text-gray-400'"
		  >
			Leech{{ sortIndicator('leechers') }}
		  </button>

		  <button
			@click="toggleSort('completed')"
			class="px-1.5 py-0.5 rounded"
			:class="sortField === 'completed' ? 'text-gray-300 bg-gray-800' : 'hover:text-gray-400'"
		  >
			Done{{ sortIndicator('completed') }}
		  </button>

		</div>
	  </div>
	</div>

    <!-- EMPTY -->
    <div v-if="store.torrents.length === 0" class="px-4 py-8 text-center text-gray-500 text-sm">
      No torrents loaded.
    </div>

    <div v-else-if="sortedTorrents.length === 0" class="px-4 py-8 text-center text-gray-500 text-sm">
      No torrents matching "{{ search }}"
    </div>

    <!-- LIST -->
    <div v-else>

      <div v-for="(group, gi) in groupedTorrents" :key="group.host">

        <!-- GROUP -->
        <div
          class="px-4 py-2 bg-gray-800/40 flex justify-between cursor-pointer hover:bg-gray-800/60"
          :class="gi > 0 ? 'border-t border-gray-800' : ''"
          @click="toggleCollapse(group.host)"
        >
          <div class="flex items-center gap-2 text-xs text-gray-400">
            <span :class="collapsedGroups.has(group.host) ? '' : 'rotate-90'">▶</span>
            <span class="text-gray-300 font-medium">{{ group.name }}</span>
            <span class="text-gray-600">{{ group.host }}</span>
          </div>
          <span class="text-xs text-gray-600">{{ group.torrents.length }}</span>
        </div>

        <!-- COLLAPSE -->
        <div
          class="grid transition-[grid-template-rows] duration-200"
          :class="collapsedGroups.has(group.host) ? 'grid-rows-[0fr]' : 'grid-rows-[1fr]'"
        >
          <div class="overflow-hidden">

            <div
              v-for="torrent in group.torrents"
              :key="torrent.infoHash"
              class="px-4 py-3 border-t border-gray-800 hover:bg-gray-800/50"
            >

              <!-- ROW 1 -->
              <div class="flex justify-between items-center gap-3">
                <div class="text-base font-medium text-white truncate">
                  {{ showFileName ? torrent.fileName : torrent.name }}
                </div>

                <span
                  class="text-xs px-2 py-0.5 rounded shrink-0"
                  :class="torrentStatus(torrent).class"
                >
                  {{ torrentStatus(torrent).label }}
                </span>
              </div>

              <!-- ROW 2 -->
              <div class="flex flex-wrap items-start sm:items-center justify-between mt-1.5 gap-y-1.5">

                <!-- STATS -->
                <div class="flex flex-wrap gap-x-3 text-[0.8rem] text-gray-500">
                  <span>{{ formatBytes(torrent.size) }}</span>

                  <span v-if="torrent.seeding || torrent.completed">
                    <span class="text-emerald-400">S:{{ torrent.seeders }}</span>
                    <span class="mx-1">/</span>
                    <span class="text-amber-400">L:{{ torrent.leechers }}</span>
                  </span>
                  <span v-else class="text-gray-600">S:-- / L:--</span>

                  <span class="text-blue-400">
                    {{ torrent.seeding && !torrent.completed ? formatSpeed(torrent.uploadRate || 0) : '--' }}
                  </span>

                  <span>Local: {{ formatBytes(torrent.uploaded) }}</span>

                  <span class="hidden sm:inline text-gray-600">
                    Reported: {{ formatBytes(torrent.reportedUploaded) }}
                  </span>
                </div>

                <!-- ACTIONS -->
                <div class="flex flex-wrap gap-2 shrink-0">

                  <button
                    v-if="torrent.active && !torrent.completed && store.status?.running"
                    @click="pauseTorrent(torrent.infoHash)"
                    :disabled="pendingActions.has(torrent.infoHash)"
                    class="text-xs text-gray-500 hover:text-amber-400 px-2.5 py-1 rounded bg-amber-500/5 border border-amber-500/10"
                  >
                    <span class="hidden sm:inline">II </span>Pause
                  </button>

                  <button
                    v-else-if="!torrent.active && !torrent.completed && store.status?.running"
                    @click="resumeTorrent(torrent.infoHash)"
                    :disabled="pendingActions.has(torrent.infoHash)"
                    class="text-xs text-gray-500 hover:text-emerald-400 px-2.5 py-1 rounded bg-emerald-500/5 border border-emerald-500/10"
                  >
                    <span class="hidden sm:inline">▶︎ </span>Resume
                  </button>

                  <button
                    v-if="!torrent.completed && store.status?.running"
                    @click="markCompleted(torrent.infoHash)"
                    :disabled="pendingActions.has(torrent.infoHash)"
                    class="text-xs text-gray-500 hover:text-cyan-400 px-2.5 py-1 rounded bg-cyan-500/5 border border-cyan-500/10"
                  >
                    <span class="hidden sm:inline">✓ </span>Done
                  </button>

                  <button
                    v-else-if="torrent.completed && store.status?.running"
                    @click="unmarkCompleted(torrent.infoHash)"
                    :disabled="pendingActions.has(torrent.infoHash)"
                    class="text-xs text-gray-500 hover:text-cyan-400 px-2.5 py-1 rounded bg-cyan-500/5 border border-cyan-500/10"
                  >
                    <span class="hidden sm:inline">↻ </span>Resume
                  </button>

                  <button
                    @click="remove(torrent.infoHash)"
                    :disabled="pendingActions.has(torrent.infoHash)"
                    class="text-xs px-2.5 py-1 rounded"
                    :class="confirmingRemove === torrent.infoHash
                      ? 'text-red-400 bg-red-500/20 border border-red-500/40'
                      : 'text-gray-500 hover:text-red-400 bg-red-500/5 border border-red-500/10'"
                  >
                    <span class="hidden sm:inline">✕ </span>
                    {{ confirmingRemove === torrent.infoHash ? 'Confirm?' : 'Remove' }}
                  </button>

                </div>
              </div>

            </div>

          </div>
        </div>

      </div>
    </div>

  </div>
</template>