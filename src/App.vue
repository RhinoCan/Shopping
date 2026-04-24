<script setup lang="ts">
import { ref, computed, nextTick, watch } from 'vue'

// ── Types ────────────────────────────────────────────────────────────────────

interface Item {
  id: number
  name: string
}

// ── Local storage helpers ────────────────────────────────────────────────────

const LS_MASTER = 'masterList'
const LS_TRIP   = 'currentTrip'

function loadMaster(): Item[] {
  try {
    return JSON.parse(localStorage.getItem(LS_MASTER) ?? '[]')
  } catch { return [] }
}

function loadTrip(): number[] {
  try {
    return JSON.parse(localStorage.getItem(LS_TRIP) ?? '[]')
  } catch { return [] }
}

function saveMaster(list: Item[]) {
  localStorage.setItem(LS_MASTER, JSON.stringify(list))
}

function saveTrip(trip: number[]) {
  localStorage.setItem(LS_TRIP, JSON.stringify(trip))
}

// ── State ────────────────────────────────────────────────────────────────────

const masterList     = ref<Item[]>(loadMaster())
const currentTrip    = ref<number[]>(loadTrip())
const showSelectedOnly = ref(false)

// Persist on change
watch(masterList,  v => saveMaster(v), { deep: true })
watch(currentTrip, v => saveTrip(v),  { deep: true })

// ── Derived ──────────────────────────────────────────────────────────────────

const displayList = computed<Item[]>(() =>
  showSelectedOnly.value
    ? masterList.value.filter(i => currentTrip.value.includes(i.id))
    : masterList.value
)

function isSelected(id: number) {
  return currentTrip.value.includes(id)
}

// ── Trip actions ─────────────────────────────────────────────────────────────

function toggleItem(id: number) {
  const idx = currentTrip.value.indexOf(id)
  if (idx === -1) currentTrip.value.push(id)
  else            currentTrip.value.splice(idx, 1)
}

function clearTrip() {
  currentTrip.value = []
}

// ── Reorder ──────────────────────────────────────────────────────────────────

function moveUp(index: number) {
  if (index === 0) return
  const list = masterList.value
  ;[list[index - 1], list[index]] = [list[index], list[index - 1]]
}

function moveDown(index: number) {
  const list = masterList.value
  if (index === list.length - 1) return
  ;[list[index], list[index + 1]] = [list[index + 1], list[index]]
}

// ── Delete ───────────────────────────────────────────────────────────────────

function deleteItem(id: number) {
  masterList.value = masterList.value.filter(i => i.id !== id)
  currentTrip.value = currentTrip.value.filter(t => t !== id)
}

// ── Inline edit ──────────────────────────────────────────────────────────────

const editingId   = ref<number | null>(null)
const editDraft   = ref('')
const editInputRef = ref<HTMLInputElement | null>(null)

function startEdit(item: Item) {
  editingId.value  = item.id
  editDraft.value  = item.name
  nextTick(() => editInputRef.value?.select())
}

function commitEdit(item: Item) {
  const trimmed = editDraft.value.trim()
  if (trimmed) item.name = trimmed
  editingId.value = null
}

function cancelEdit() {
  editingId.value = null
}

// Long-tap support for touch devices
let longTapTimer: ReturnType<typeof setTimeout> | null = null

function startLongTap(item: Item) {
  longTapTimer = setTimeout(() => startEdit(item), 500)
}

function cancelLongTap() {
  if (longTapTimer !== null) {
    clearTimeout(longTapTimer)
    longTapTimer = null
  }
}

// ── Add dialog ───────────────────────────────────────────────────────────────

const addDialog = ref({ open: false, afterId: null as number | null, draft: '' })
const addInputRef = ref<HTMLInputElement | null>(null)

function openAddDialog(afterId: number | null) {
  addDialog.value = { open: true, afterId, draft: '' }
  nextTick(() => addInputRef.value?.focus())
}

function confirmAdd() {
  const name = addDialog.value.draft.trim()
  if (!name) return

  const newItem: Item = { id: Date.now(), name }

  if (addDialog.value.afterId === null) {
    // Append to end
    masterList.value.push(newItem)
  } else {
    const idx = masterList.value.findIndex(i => i.id === addDialog.value.afterId)
    masterList.value.splice(idx + 1, 0, newItem)
  }

  addDialog.value.open = false
}

// ── Export / Import ──────────────────────────────────────────────────────────

function exportList() {
  const payload = {
    masterList: masterList.value,
    currentTrip: currentTrip.value,
  }
  const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' })
  const url  = URL.createObjectURL(blob)
  const a    = document.createElement('a')
  a.href     = url
  a.download = 'shopping-list.json'
  a.click()
  URL.revokeObjectURL(url)
}

function importList(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const data = JSON.parse(e.target?.result as string)
      if (!Array.isArray(data.masterList)) throw new Error('Invalid format')
      masterList.value  = data.masterList
      currentTrip.value = Array.isArray(data.currentTrip) ? data.currentTrip : []
    } catch {
      alert('Could not import — file doesn\'t look like a shopping list export.')
    }
    // Reset the input so the same file can be re-imported if needed
    ;(event.target as HTMLInputElement).value = ''
  }
  reader.readAsText(file)
}
</script>

<template>
  <v-app :theme="'light'" class="app-root">
    <v-main class="main-area">
      <div class="list-container">

        <!-- ── Header ── -->
        <header class="app-header">
          <div class="header-top">
            <span class="app-title">Shopping</span>
            <span class="item-count">{{ masterList.length }} items / {{ currentTrip.length }} selected </span>
          </div>
          <div class="header-controls">
            <v-btn-toggle v-model="showSelectedOnly" mandatory density="compact" class="toggle-group">
              <v-btn :value="false">All</v-btn>
              <v-btn :value="true">Selected</v-btn>
            </v-btn-toggle>
            <!-- <button
              class="ctrl-btn toggle-btn"
              :class="{ active: showSelectedOnly }"
              @click="showSelectedOnly = !showSelectedOnly"
            >
              {{ showSelectedOnly ? 'Selected Only' : 'Show All' }}
            </button> -->
            <button
              class="ctrl-btn clear-btn"
              :disabled="currentTrip.length === 0"
              @click="clearTrip"
            >
              Clear Trip
            </button>
            <button
              class="ctrl-btn io-btn"
              :disabled="masterList.length === 0"
              @click="exportList"
            >
              Export
            </button>
            <label class="ctrl-btn io-btn import-label">
              Import
              <input type="file" accept=".json" class="file-input-hidden" @change="importList" />
            </label>
          </div>
        </header>

        <!-- ── Divider ── -->
        <div class="header-rule" />

        <!-- ── Empty state ── -->
        <div v-if="displayList.length === 0" class="empty-state">
          <span v-if="showSelectedOnly">Nothing selected for this trip.</span>
          <span v-else>No items yet. Add your first one below.</span>
          <button v-if="!showSelectedOnly" class="add-first-btn" @click="openAddDialog(null)">
            + Add item
          </button>
        </div>

        <!-- ── List ── -->
        <TransitionGroup name="row" tag="ul" class="item-list">
          <li
            v-for="(item, index) in displayList"
            :key="item.id"
            class="item-row"
            :class="{ checked: isSelected(item.id) }"
          >
            <!-- Checkbox -->
            <button
              class="check-btn"
              :aria-label="isSelected(item.id) ? 'Deselect' : 'Select'"
              @click="toggleItem(item.id)"
            >
              <svg class="check-icon" viewBox="0 0 18 18">
                <rect class="check-box" x="1" y="1" width="16" height="16" rx="3" />
                <polyline class="check-mark" points="3.5,9 7.5,13 14.5,5" />
              </svg>
            </button>

            <!-- Item name — dblclick / long-tap to edit inline -->
            <span
              v-if="editingId !== item.id"
              class="item-name"
              :class="{ 'item-name--checked': isSelected(item.id) }"
              @dblclick="!showSelectedOnly && startEdit(item)"
              @touchstart="!showSelectedOnly && startLongTap(item)"
              @touchend="cancelLongTap"
              @touchmove="cancelLongTap"
            >{{ item.name }}</span>

            <input
              v-else
              :ref="el => { if (el) editInputRef = el as HTMLInputElement }"
              v-model="editDraft"
              class="item-edit-input"
              type="text"
              @keydown.enter="commitEdit(item)"
              @keydown.escape="cancelEdit"
              @blur="commitEdit(item)"
            />

            <!-- Show-All controls -->
            <template v-if="!showSelectedOnly">
              <div class="row-controls">
                <button
                  class="icon-btn"
                  :disabled="index === 0"
                  aria-label="Move up"
                  @click="moveUp(index)"
                >▲</button>
                <button
                  class="icon-btn"
                  :disabled="index === displayList.length - 1"
                  aria-label="Move down"
                  @click="moveDown(index)"
                >▼</button>
                <button
                  class="icon-btn add-row-btn"
                  aria-label="Add item below"
                  @click="openAddDialog(item.id)"
                >＋</button>
                <button
                  class="icon-btn del-btn"
                  aria-label="Delete item"
                  @click="deleteItem(item.id)"
                >✕</button>
              </div>
            </template>
          </li>
        </TransitionGroup>

        <!-- ── Add-to-bottom button (Show All, list non-empty) ── -->
        <div v-if="!showSelectedOnly && masterList.length > 0" class="add-bottom-row">
          <button class="add-bottom-btn" @click="openAddDialog(null)">+ Add item</button>
        </div>

      </div>
    </v-main>

    <!-- ── Add Item Dialog ── -->
    <v-dialog v-model="addDialog.open" max-width="360" persistent>
      <div class="dialog-card">
        <p class="dialog-title">New item</p>
        <input
          ref="addInputRef"
          v-model="addDialog.draft"
          class="dialog-input"
          type="text"
          placeholder="e.g. Milk 2% Lactantia 4L"
          @keydown.enter="confirmAdd"
          @keydown.escape="addDialog.open = false"
        />
        <div class="dialog-actions">
          <button class="dialog-btn cancel" @click="addDialog.open = false">Cancel</button>
          <button class="dialog-btn confirm" :disabled="!addDialog.draft.trim()" @click="confirmAdd">Add</button>
        </div>
      </div>
    </v-dialog>

  </v-app>
</template>

<style scoped>
/* ── Fonts ── */
@import url('https://fonts.googleapis.com/css2?family=Zilla+Slab:wght@600&family=DM+Sans:wght@400;500&display=swap');

/* ── Root / layout ── */
.app-root {
  background: #f5f0e8 !important;
  font-family: 'DM Sans', sans-serif;
  min-height: 100dvh;
}

.main-area {
  display: flex;
  justify-content: center;
  padding: 0 !important;
}

.list-container {
  width: 100%;
  max-width: 540px;
  padding: 28px 20px 80px;
}

/* ── Header ── */
.app-header {
  margin-bottom: 0;
}

.header-top {
  display: flex;
  align-items: baseline;
  gap: 12px;
  margin-bottom: 14px;
}

.app-title {
  font-family: 'Zilla Slab', serif;
  font-size: 2rem;
  font-weight: 600;
  color: #1a1a1a;
  letter-spacing: -0.02em;
  line-height: 1;
}

.item-count {
  font-size: 0.78rem;
  color: #888;
  letter-spacing: 0.04em;
  font-weight: 500;
  text-transform: uppercase;
}

.header-controls {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
  flex-wrap: wrap;
  align-items: center;
}

.toggle-group {
  border: 1.5px solid #c8bfb0;
  border-radius: 6px;
}

.toggle-group .v-btn--active {
  background-color: #2d2d2d !important;
  color: #f5f0e8 !important;
}

.toggle-group .v-btn:not(.v-btn--active) {
  background-color: #f5f0e8 !important;
  color: #888 !important;
}

.ctrl-btn {
  font-family: 'DM Sans', sans-serif;
  font-size: 0.82rem;
  font-weight: 500;
  letter-spacing: 0.01em;
  padding: 6px 14px;
  border-radius: 6px;
  border: 1.5px solid #c8bfb0;
  cursor: pointer;
  transition: background 0.15s, color 0.15s, border-color 0.15s;
  background: transparent;
  color: #444;
}

.ctrl-btn:hover:not(:disabled) {
  background: #e8e0d0;
}

.ctrl-btn:disabled {
  opacity: 0.38;
  cursor: default;
}

.toggle-btn.active {
  background: #2d2d2d;
  color: #f5f0e8;
  border-color: #2d2d2d;
}

.clear-btn {
  color: #b04030;
  border-color: #d4a098;
}

.clear-btn:hover:not(:disabled) {
  background: #f5ddd8;
  border-color: #b04030;
}

.io-btn {
  color: #3a5a3a;
  border-color: #a8c0a8;
}

.io-btn:hover:not(:disabled) {
  background: #ddeadd;
  border-color: #3a5a3a;
}

.import-label {
  cursor: pointer;
  display: inline-flex;
  align-items: center;
}

.file-input-hidden {
  display: none;
}

.header-rule {
  height: 2px;
  background: #d6cfc4;
  margin-bottom: 4px;
}

/* ── Empty state ── */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 12px;
  padding: 32px 0 0;
  color: #888;
  font-size: 0.92rem;
}

.add-first-btn {
  font-family: 'DM Sans', sans-serif;
  font-size: 0.88rem;
  font-weight: 500;
  color: #2d2d2d;
  background: transparent;
  border: 1.5px dashed #c8bfb0;
  border-radius: 6px;
  padding: 7px 16px;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s;
}

.add-first-btn:hover {
  background: #ece6da;
  border-color: #999;
}

/* ── List ── */
.item-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.item-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 9px 0;
  border-bottom: 1px solid #e0d9cf;
  min-height: 44px;
  transition: background 0.1s;
}

.item-row:last-child {
  border-bottom: none;
}

/* ── Checkbox ── */
.check-btn {
  flex-shrink: 0;
  background: none;
  border: none;
  padding: 2px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.check-icon {
  width: 20px;
  height: 20px;
  overflow: visible;
}

.check-box {
  fill: #f5f0e8;
  stroke: #b0a898;
  stroke-width: 1.5;
  transition: fill 0.18s, stroke 0.18s;
}

.check-mark {
  fill: none;
  stroke: #f5f0e8;
  stroke-width: 2;
  stroke-linecap: round;
  stroke-linejoin: round;
  transition: stroke 0.18s;
}

.checked .check-box {
  fill: #2d2d2d;
  stroke: #2d2d2d;
}

.checked .check-mark {
  stroke: #f5f0e8;
}

/* ── Item name ── */
.item-name {
  flex: 1;
  font-size: 0.95rem;
  color: #1a1a1a;
  line-height: 1.4;
  cursor: text;
  user-select: none;
  transition: color 0.18s;
}

.item-name--checked {
  color: #1a3a5c;
  font-weight: 700;
  font-size: 1rem;
}

.item-edit-input {
  flex: 1;
  font-family: 'DM Sans', sans-serif;
  font-size: 0.95rem;
  color: #1a1a1a;
  background: #ece6da;
  border: 1.5px solid #b0a898;
  border-radius: 4px;
  padding: 3px 8px;
  outline: none;
}

.item-edit-input:focus {
  border-color: #2d2d2d;
}

/* ── Row controls ── */
.row-controls {
  display: flex;
  gap: 2px;
  flex-shrink: 0;
}

.icon-btn {
  background: none;
  border: none;
  cursor: pointer;
  color: #999;
  font-size: 0.72rem;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  transition: background 0.12s, color 0.12s;
  padding: 0;
}

.icon-btn:hover:not(:disabled) {
  background: #e0d9cf;
  color: #2d2d2d;
}

.icon-btn:disabled {
  opacity: 0.22;
  cursor: default;
}

.add-row-btn {
  font-size: 1rem;
  color: #5a7a5a;
}

.add-row-btn:hover:not(:disabled) {
  background: #ddeadd;
  color: #2d5a2d;
}

.del-btn {
  color: #a06050;
  font-size: 0.75rem;
}

.del-btn:hover:not(:disabled) {
  background: #f0ddd8;
  color: #7a2a1a;
}

/* ── Add-to-bottom ── */
.add-bottom-row {
  padding: 12px 0 0;
}

.add-bottom-btn {
  font-family: 'DM Sans', sans-serif;
  font-size: 0.85rem;
  font-weight: 500;
  color: #666;
  background: transparent;
  border: 1.5px dashed #c8bfb0;
  border-radius: 6px;
  padding: 7px 16px;
  cursor: pointer;
  width: 100%;
  text-align: left;
  transition: border-color 0.15s, background 0.15s, color 0.15s;
}

.add-bottom-btn:hover {
  background: #ece6da;
  border-color: #999;
  color: #2d2d2d;
}

/* ── TransitionGroup ── */
.row-enter-active,
.row-leave-active {
  transition: opacity 0.2s, transform 0.2s;
}

.row-enter-from {
  opacity: 0;
  transform: translateY(-6px);
}

.row-leave-to {
  opacity: 0;
  transform: translateX(12px);
}

/* ── Dialog ── */
.dialog-card {
  background: #f5f0e8;
  border-radius: 10px;
  padding: 24px 22px 20px;
  border: 1.5px solid #d6cfc4;
}

.dialog-title {
  font-family: 'Zilla Slab', serif;
  font-size: 1.15rem;
  font-weight: 600;
  color: #1a1a1a;
  margin: 0 0 14px;
}

.dialog-input {
  width: 100%;
  font-family: 'DM Sans', sans-serif;
  font-size: 0.95rem;
  color: #1a1a1a;
  background: #ece6da;
  border: 1.5px solid #c0b8ae;
  border-radius: 6px;
  padding: 9px 12px;
  outline: none;
  box-sizing: border-box;
  transition: border-color 0.15s;
}

.dialog-input:focus {
  border-color: #2d2d2d;
}

.dialog-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 16px;
}

.dialog-btn {
  font-family: 'DM Sans', sans-serif;
  font-size: 0.85rem;
  font-weight: 500;
  padding: 7px 18px;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.12s, color 0.12s;
}

.dialog-btn.cancel {
  background: transparent;
  border: 1.5px solid #c8bfb0;
  color: #555;
}

.dialog-btn.cancel:hover {
  background: #e8e0d0;
}

.dialog-btn.confirm {
  background: #2d2d2d;
  border: 1.5px solid #2d2d2d;
  color: #f5f0e8;
}

.dialog-btn.confirm:hover:not(:disabled) {
  background: #111;
}

.dialog-btn.confirm:disabled {
  opacity: 0.35;
  cursor: default;
}
</style>
