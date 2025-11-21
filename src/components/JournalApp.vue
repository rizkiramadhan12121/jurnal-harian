<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const storageKey = 'journalEntries'
const storageKeyEnc = 'journalEntriesEnc'
const authKey = 'journalAuth'

const entries = ref([])
const searchQuery = ref('')
const newEntryDate = ref('')
const newEntryTitle = ref('')
const newEntryContent = ref('')
const newEntryTags = ref('')
const editingId = ref(null)
const editDate = ref('')
const editTitle = ref('')
const editContent = ref('')
const editTags = ref('')
const filterFromDate = ref('')
const filterToDate = ref('')
const sortBy = ref('date')
const sortDir = ref('desc')
const toastMsg = ref('')
const toastVisible = ref(false)
const locked = ref(false)
const passwordSet = ref(false)
const passwordInput = ref('')
const newPasswordInput = ref('')
const cryptoSalt = ref('')
let currentPassword = ''

onMounted(() => {
  try {
    const auth = localStorage.getItem(authKey)
    if (auth) {
      const a = JSON.parse(auth)
      cryptoSalt.value = a.salt || ''
    }
    const enc = localStorage.getItem(storageKeyEnc)
    if (enc) {
      passwordSet.value = true
      locked.value = true
    } else {
      const raw = localStorage.getItem(storageKey)
      if (raw) {
        const parsed = JSON.parse(raw)
        if (Array.isArray(parsed)) entries.value = parsed
      }
    }
  } catch (e) {}
})

watch(
  entries,
  async (val) => {
    try {
      if (passwordSet.value && currentPassword) {
        await saveEncrypted()
      } else if (!passwordSet.value) {
        localStorage.setItem(storageKey, JSON.stringify(val))
      }
    } catch (e) {}
  },
  { deep: true }
)

const sortedEntries = computed(() => {
  return [...entries.value]
    .filter((e) => !e.archived)
    .sort((a, b) => {
      if (!!a.pinned !== !!b.pinned) return a.pinned ? -1 : 1
      let cmp = 0
      if (sortBy.value === 'date') {
        cmp = a.date.localeCompare(b.date)
      } else if (sortBy.value === 'title') {
        cmp = a.title.localeCompare(b.title)
      } else {
        cmp = a.id - b.id
      }
      return sortDir.value === 'asc' ? cmp : -cmp
    })
})

const filteredEntries = computed(() => {
  const q = searchQuery.value.toLowerCase().trim()
  return sortedEntries.value.filter((e) => {
    const inRange = (!filterFromDate.value || e.date >= filterFromDate.value) && (!filterToDate.value || e.date <= filterToDate.value)
    if (!inRange) return false
    if (!q) return true
    const tags = Array.isArray(e.tags) ? e.tags.join(' ').toLowerCase() : ''
    return e.title.toLowerCase().includes(q) || e.content.toLowerCase().includes(q) || tags.includes(q)
  })
})

const groupedEntries = computed(() => {
  const pinned = filteredEntries.value.filter((e) => e.pinned)
  const rest = filteredEntries.value.filter((e) => !e.pinned)
  const groups = []
  if (pinned.length) groups.push({ date: 'Tersemat', items: pinned, pinned: true })
  const map = new Map()
  for (const e of rest) {
    const list = map.get(e.date) || []
    list.push(e)
    map.set(e.date, list)
  }
  const dateGroups = Array.from(map.entries()).map(([date, items]) => ({ date, items }))
  return [...groups, ...dateGroups]
})

function formatDate(d) {
  try {
    return new Date(d).toLocaleDateString('id-ID', { day: 'numeric', month: 'long', year: 'numeric' })
  } catch (e) {
    return d
  }
}

const todayCount = computed(() => {
  const today = new Date().toISOString().slice(0, 10)
  return entries.value.filter((e) => e.date === today).length
})

function resetNewForm() {
  newEntryDate.value = ''
  newEntryTitle.value = ''
  newEntryContent.value = ''
  newEntryTags.value = ''
}

function addEntry() {
  const date = newEntryDate.value.trim()
  const title = newEntryTitle.value.trim()
  const content = newEntryContent.value.trim()
  const tags = newEntryTags.value.split(',').map((t) => t.trim()).filter((t) => t)
  if (!date || !title || !content) return
  entries.value.push({ id: Date.now(), date, title, content, tags, pinned: false, archived: false })
  resetNewForm()
  showToast('Catatan ditambahkan')
}

function startEdit(entry) {
  editingId.value = entry.id
  editDate.value = entry.date
  editTitle.value = entry.title
  editContent.value = entry.content
  editTags.value = Array.isArray(entry.tags) ? entry.tags.join(', ') : ''
}

function cancelEdit() {
  editingId.value = null
}

function saveEdit(id) {
  const date = editDate.value.trim()
  const title = editTitle.value.trim()
  const content = editContent.value.trim()
  const tags = editTags.value.split(',').map((t) => t.trim()).filter((t) => t)
  if (!date || !title || !content) return
  const idx = entries.value.findIndex((e) => e.id === id)
  if (idx !== -1) {
    entries.value[idx] = { ...entries.value[idx], date, title, content, tags }
  }
  editingId.value = null
  showToast('Perubahan disimpan')
}

function deleteEntry(id) {
  entries.value = entries.value.filter((e) => e.id !== id)
  showToast('Catatan dihapus')
}

function togglePin(entry) {
  const idx = entries.value.findIndex((e) => e.id === entry.id)
  if (idx !== -1) {
    entries.value[idx] = { ...entries.value[idx], pinned: !entries.value[idx].pinned }
    showToast(entries.value[idx].pinned ? 'Catatan disematkan' : 'Catatan dilepas')
  }
}

function archiveEntry(id) {
  const idx = entries.value.findIndex((e) => e.id === id)
  if (idx !== -1) {
    entries.value[idx] = { ...entries.value[idx], archived: true, pinned: false }
    showToast('Diarsipkan')
  }
}

function restoreEntry(id) {
  const idx = entries.value.findIndex((e) => e.id === id)
  if (idx !== -1) {
    entries.value[idx] = { ...entries.value[idx], archived: false }
    showToast('Dipulihkan')
  }
}

function deleteForever(id) {
  entries.value = entries.value.filter((e) => e.id !== id)
  showToast('Dihapus permanen')
}

const archivedEntries = computed(() => entries.value.filter((e) => e.archived))

function getRandomBytes(n) {
  const a = new Uint8Array(n)
  crypto.getRandomValues(a)
  return a
}

function bufToBase64(buf) {
  const bytes = new Uint8Array(buf)
  let binary = ''
  for (let i = 0; i < bytes.length; i++) binary += String.fromCharCode(bytes[i])
  return btoa(binary)
}

function base64ToBuf(b64) {
  const binary = atob(b64)
  const bytes = new Uint8Array(binary.length)
  for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i)
  return bytes.buffer
}

async function deriveKey(password, salt) {
  const enc = new TextEncoder().encode(password)
  const keyMaterial = await crypto.subtle.importKey('raw', enc, 'PBKDF2', false, ['deriveKey'])
  return crypto.subtle.deriveKey(
    { name: 'PBKDF2', salt, iterations: 100000, hash: 'SHA-256' },
    keyMaterial,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  )
}

async function saveEncrypted() {
  let salt = cryptoSalt.value ? base64ToBuf(cryptoSalt.value) : getRandomBytes(16)
  const key = await deriveKey(currentPassword, salt)
  const iv = getRandomBytes(12)
  const data = new TextEncoder().encode(JSON.stringify(entries.value))
  const ct = await crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, data)
  localStorage.setItem(authKey, JSON.stringify({ salt: bufToBase64(salt) }))
  cryptoSalt.value = bufToBase64(salt)
  localStorage.setItem(storageKeyEnc, JSON.stringify({ iv: bufToBase64(iv), ct: bufToBase64(ct) }))
  localStorage.removeItem(storageKey)
}

async function decryptEntries(password) {
  const auth = localStorage.getItem(authKey)
  const enc = localStorage.getItem(storageKeyEnc)
  if (!auth || !enc) return []
  const { salt } = JSON.parse(auth)
  const { iv, ct } = JSON.parse(enc)
  const key = await deriveKey(password, base64ToBuf(salt))
  const pt = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv: new Uint8Array(base64ToBuf(iv)) },
    key,
    base64ToBuf(ct)
  )
  const json = new TextDecoder().decode(pt)
  return JSON.parse(json)
}

async function setPassword() {
  if (!newPasswordInput.value) return
  currentPassword = newPasswordInput.value
  passwordSet.value = true
  locked.value = true
  entries.value = []
  await saveEncrypted()
  showToast('Password diaktifkan')
}

async function unlock() {
  if (!passwordInput.value) return
  try {
    const data = await decryptEntries(passwordInput.value)
    entries.value = Array.isArray(data) ? data : []
    currentPassword = passwordInput.value
    locked.value = false
    showToast('Dibuka')
  } catch (e) {
    showToast('Password salah')
  }
}

function lock() {
  currentPassword = ''
  locked.value = true
  entries.value = []
  showToast('Dikunci')
}

async function changePassword() {
  if (!newPasswordInput.value) return
  currentPassword = newPasswordInput.value
  await saveEncrypted()
  showToast('Password diganti')
}

const syncUrl = ref('')
const syncMerge = ref(true)

async function syncUpload() {
  if (!syncUrl.value) return
  try {
    const res = await fetch(syncUrl.value, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(entries.value),
    })
    showToast(res.ok ? 'Sinkron upload sukses' : 'Sinkron upload gagal')
  } catch (e) {
    showToast('Sinkron upload error')
  }
}

async function syncDownload() {
  if (!syncUrl.value) return
  try {
    const res = await fetch(syncUrl.value)
    const data = await res.json()
    if (Array.isArray(data)) {
      if (syncMerge.value) {
        const map = new Map(entries.value.map((e) => [e.id, e]))
        for (const e of data) map.set(e.id, e)
        entries.value = Array.from(map.values())
      } else {
        entries.value = data
      }
      showToast('Sinkron download sukses')
    } else {
      showToast('Format sinkron tidak valid')
    }
  } catch (e) {
    showToast('Sinkron download error')
  }
}
</script>

<template>
  <div class="page">
    <header class="header">
      <div class="header-main">
        <h1>Jurnal Harian</h1>
        <p class="subtitle">Tema putih, rapi, modern</p>
      </div>
      <div class="header-actions">
        <div class="filters">
          <input class="search" type="text" v-model="searchQuery" placeholder="Cari judul/isi/tag" />
          <div class="dates">
            <input type="date" v-model="filterFromDate" />
            <span>→</span>
            <input type="date" v-model="filterToDate" />
          </div>
          <div class="sorts">
            <select v-model="sortBy">
              <option value="date">Tanggal</option>
              <option value="title">Judul</option>
              <option value="id">Terbaru</option>
            </select>
            <select v-model="sortDir">
              <option value="desc">↓</option>
              <option value="asc">↑</option>
            </select>
          </div>
        </div>
        <div class="chips">
          <span class="chip">Total: {{ entries.length }}</span>
          <span class="chip">Hari ini: {{ todayCount }}</span>
        </div>
        <div class="header-buttons">
          <button class="outline" @click="exportJSON">Ekspor JSON</button>
          <input id="importInput" type="file" accept="application/json" @change="importJSON" hidden />
          <button class="outline" @click="() => document.getElementById('importInput').click()">Impor JSON</button>
          <button class="danger" @click="clearAll">Bersihkan</button>
        </div>
        <div class="protect">
          <template v-if="!passwordSet">
            <input type="password" v-model="newPasswordInput" placeholder="Set password" />
            <button class="outline" @click="setPassword">Aktifkan</button>
          </template>
          <template v-else>
            <template v-if="locked">
              <input type="password" v-model="passwordInput" placeholder="Masukkan password" />
              <button @click="unlock">Buka</button>
            </template>
            <template v-else>
              <input type="password" v-model="newPasswordInput" placeholder="Ganti password" />
              <button class="outline" @click="changePassword">Ganti</button>
              <button class="danger" @click="lock">Kunci</button>
            </template>
          </template>
        </div>
        <div class="sync">
          <input type="text" v-model="syncUrl" placeholder="URL sinkron" />
          <label class="merge"><input type="checkbox" v-model="syncMerge" /> Gabung</label>
          <button class="outline" @click="syncUpload">Upload</button>
          <button class="outline" @click="syncDownload">Download</button>
        </div>
      </div>
    </header>

    <section class="card">
      <h2>Tambah Catatan</h2>
      <div class="form">
        <label>
          Tanggal
          <input type="date" v-model="newEntryDate" />
        </label>
        <label>
          Judul
          <input type="text" v-model="newEntryTitle" placeholder="Judul catatan" />
        </label>
        <label>
          Isi
          <textarea v-model="newEntryContent" rows="4" placeholder="Tulis catatan"></textarea>
        </label>
        <label>
          Tag
          <input type="text" v-model="newEntryTags" placeholder="Pisahkan dengan koma" />
        </label>
        <div class="actions">
          <button @click="addEntry">Simpan</button>
          <button class="secondary" @click="resetNewForm">Reset</button>
        </div>
      </div>
    </section>

    <section class="list">
      <h2>Daftar Catatan</h2>
      <div v-if="groupedEntries.length === 0" class="empty">Belum ada catatan</div>
      <div v-else class="groups">
        <div v-for="group in groupedEntries" :key="group.date" class="group">
          <div class="group-header">
            <div class="group-date">{{ group.pinned ? 'Tersemat' : formatDate(group.date) }}</div>
            <div class="group-count">{{ group.items.length }} catatan</div>
          </div>
          <ul class="items">
            <li v-for="entry in group.items" :key="entry.id" class="item">
              <div v-if="editingId !== entry.id" class="view">
                <div class="meta">
                  <span class="title">{{ entry.title }}</span>
                  <span v-if="entry.pinned" class="badge">Tersemat</span>
                </div>
                <p class="content">{{ entry.content }}</p>
                <div class="tags" v-if="entry.tags && entry.tags.length">
                  <span v-for="t in entry.tags" :key="t" class="tag">#{{ t }}</span>
                </div>
                <div class="actions">
                  <button class="outline" @click="togglePin(entry)">{{ entry.pinned ? 'Lepas' : 'Semat' }}</button>
                  <button @click="startEdit(entry)">Edit</button>
                  <button class="danger" @click="deleteEntry(entry.id)">Hapus</button>
                  <button class="outline" @click="archiveEntry(entry.id)">Arsipkan</button>
                </div>
              </div>
              <div v-else class="edit">
                <div class="form inline">
                  <label>
                    Tanggal
                    <input type="date" v-model="editDate" />
                  </label>
                  <label>
                    Judul
                    <input type="text" v-model="editTitle" />
                  </label>
                  <label>
                    Isi
                    <textarea v-model="editContent" rows="4"></textarea>
                  </label>
                  <label>
                    Tag
                    <input type="text" v-model="editTags" />
                  </label>
                  <div class="actions">
                    <button @click="saveEdit(entry.id)">Simpan</button>
                    <button class="secondary" @click="cancelEdit">Batal</button>
                  </div>
                </div>
              </div>
            </li>
          </ul>
        </div>
      </div>
    </section>

    <section class="list">
      <h2>Arsip</h2>
      <div v-if="archivedEntries.length === 0" class="empty">Tidak ada arsip</div>
      <ul v-else class="items">
        <li v-for="entry in archivedEntries" :key="entry.id" class="item">
          <div class="view">
            <div class="meta">
              <span class="title">{{ entry.title }}</span>
              <span class="badge">Arsip</span>
            </div>
            <p class="content">{{ entry.content }}</p>
            <div class="tags" v-if="entry.tags && entry.tags.length">
              <span v-for="t in entry.tags" :key="t" class="tag">#{{ t }}</span>
            </div>
            <div class="actions">
              <button @click="restoreEntry(entry.id)">Pulihkan</button>
              <button class="danger" @click="deleteForever(entry.id)">Hapus Permanen</button>
            </div>
          </div>
        </li>
      </ul>
    </section>
  </div>

  <div v-if="toastVisible" class="toast">{{ toastMsg }}</div>
</template>

<style scoped>
.page {
  display: grid;
  gap: 1.5rem;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: end;
}

.subtitle {
  color: var(--muted);
}

.header-actions {
  display: grid;
  gap: 0.75rem;
  justify-items: end;
}

.filters {
  display: flex;
  gap: 0.5rem;
}

.dates {
  display: flex;
  gap: 0.35rem;
  align-items: center;
}

.sorts {
  display: flex;
  gap: 0.35rem;
}

.search {
  width: 280px;
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 0.55rem 0.75rem;
  background: var(--card-bg);
}

.chips {
  display: flex;
  gap: 0.5rem;
}

.header-buttons {
  display: flex;
  gap: 0.5rem;
}

.chip {
  background: var(--card-bg);
  border: 1px solid var(--border);
  color: var(--muted);
  border-radius: 999px;
  padding: 0.25rem 0.6rem;
  font-size: 0.85rem;
}

.outline {
  background: var(--card-bg);
  color: var(--text);
}

.card {
  background: var(--card-bg);
  border: 1px solid var(--border);
  border-radius: 16px;
  padding: 1rem;
  box-shadow: 0 10px 20px rgba(16, 24, 40, 0.06);
}

.form {
  display: grid;
  gap: 0.75rem;
}

.form label {
  display: grid;
  gap: 0.25rem;
}

input,
textarea {
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 0.6rem 0.75rem;
  font-size: 1rem;
  background: var(--card-bg);
  color: var(--text);
}

.actions {
  display: flex;
  gap: 0.5rem;
}

.secondary {
  background-color: #f3f4f6;
}

.danger {
  background-color: #ef4444;
  color: #fff;
}

.list .groups {
  display: grid;
  gap: 1rem;
}

.group {
  background: transparent;
}

.group-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  color: var(--muted);
}

.items {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1rem;
}

.item {
  border: 1px solid var(--border);
  border-radius: 16px;
  padding: 1rem;
  background: var(--card-bg);
  box-shadow: 0 6px 16px rgba(16, 24, 40, 0.06);
}

.badge {
  border: 1px solid var(--border);
  color: var(--muted);
  border-radius: 999px;
  padding: 0.1rem 0.5rem;
  font-size: 0.8rem;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.3rem;
}

.tag {
  background: #eef2ff;
  color: #3730a3;
  border-radius: 999px;
  padding: 0.15rem 0.5rem;
  font-size: 0.8rem;
}

.toast {
  position: fixed;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
  background: var(--text);
  color: #fff;
  padding: 0.5rem 0.75rem;
  border-radius: 8px;
}

.meta {
  display: flex;
  gap: 0.75rem;
  align-items: baseline;
}

.title {
  font-weight: 600;
}

.content {
  margin: 0.5rem 0 0.75rem;
}

.empty {
  color: var(--muted);
}

.inline label {
  grid-template-columns: 100%;
}
</style>
