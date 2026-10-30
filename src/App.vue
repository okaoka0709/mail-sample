<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'
import Icon from './components/Icon.vue'

const STORAGE = 'roundmail-demo-v1'
const account = { name: '林小夏', email: 'xia@example.com' }
const folders = [
  { id: 'inbox', label: '收件匣', icon: 'inbox', color: '#28abe0' },
  { id: 'drafts', label: '草稿匣', icon: 'draft', color: '#087bb7' },
  { id: 'sent', label: '寄件備份', icon: 'sent', color: '#00a5a6' },
  { id: 'spam', label: '垃圾郵件', icon: 'spam', color: '#8ec41b' },
  { id: 'trash', label: '垃圾桶', icon: 'trash', color: '#9ba2a7' },
]
const mails = ref([])
const theme = ref('classic')
const folder = ref('inbox')
const query = ref('')
const selectedId = ref(null)
const checked = ref([])
const loading = ref(true)
const error = ref('')
const notice = ref('')
const mobileReading = ref(false)
const sidebarOpen = ref(false)
const composer = ref(null)
const composeDialog = ref(null)
const composeError = ref('')
let initialMails = []
let noticeTimer

const visible = computed(() => {
  const term = query.value.trim().toLocaleLowerCase()
  return mails.value.filter(mail => mail.folder === folder.value &&
    (!term || [mail.subject, mail.from.name, mail.from.email, mail.body, mail.to].join(' ').toLocaleLowerCase().includes(term)))
    .sort((a, b) => new Date(b.date) - new Date(a.date))
})
const selected = computed(() => visible.value.find(mail => mail.id === selectedId.value))
const folderName = computed(() => folders.find(item => item.id === folder.value)?.label)
const unreadCount = computed(() => mails.value.filter(mail => mail.folder === 'inbox' && !mail.read).length)
const targets = computed(() => checked.value.length ? checked.value : selected.value ? [selected.value.id] : [])
const allChecked = computed(() => visible.value.length > 0 && checked.value.length === visible.value.length)
const selectedIndex = computed(() => visible.value.findIndex(mail => mail.id === selectedId.value))
const dateLabel = date => new Intl.DateTimeFormat('zh-TW', { month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', hour12: false }).format(new Date(date))

function notify(message) {
  notice.value = message
  clearTimeout(noticeTimer)
  noticeTimer = setTimeout(() => { notice.value = '' }, 3500)
}
function persist() {
  if (loading.value || error.value) return
  try { localStorage.setItem(STORAGE, JSON.stringify({ mails: mails.value, theme: theme.value })) }
  catch { notify('瀏覽器無法儲存資料；本次操作仍可使用。') }
}
watch([mails, theme], persist, { deep: true })
watch(composer, async draft => {
  if (draft) {
    await nextTick()
    composeDialog.value?.showModal()
  }
})
watch(visible, list => {
  checked.value = checked.value.filter(id => list.some(mail => mail.id === id))
  if (!list.some(mail => mail.id === selectedId.value)) selectedId.value = list[0]?.id ?? null
})

async function load() {
  loading.value = true
  error.value = ''
  try {
    const response = await fetch(`${import.meta.env.BASE_URL}data/mail.json`)
    if (!response.ok) throw new Error('load')
    initialMails = await response.json()
    let saved
    try { saved = JSON.parse(localStorage.getItem(STORAGE)) } catch { /* Use the JSON seed if storage is unavailable. */ }
    const valid = Array.isArray(saved?.mails) && saved.mails.every(mail =>
      mail && typeof mail.id === 'string' && folders.some(item => item.id === mail.folder) &&
      typeof mail.subject === 'string' && typeof mail.body === 'string' && typeof mail.to === 'string' &&
      typeof mail.from?.name === 'string' && typeof mail.from?.email === 'string' &&
      Number.isFinite(Date.parse(mail.date)) && Array.isArray(mail.attachments))
    mails.value = structuredClone(valid ? saved.mails : initialMails)
    theme.value = saved?.theme === 'fresh' ? 'fresh' : 'classic'
    selectedId.value = mails.value.find(mail => mail.folder === 'inbox')?.id ?? null
  } catch { error.value = '無法載入示範信件，請確認以本機或網站網址開啟後重試。' }
  finally { loading.value = false }
}
onMounted(load)

function changeFolder(id) {
  folder.value = id
  checked.value = []
  mobileReading.value = false
  sidebarOpen.value = false
}
function openMail(mail) {
  selectedId.value = mail.id
  mail.read = true
  mobileReading.value = true
}
function toggleAll() { checked.value = allChecked.value ? [] : visible.value.map(mail => mail.id) }
function apply(action) {
  mails.value.filter(mail => targets.value.includes(mail.id)).forEach(action)
}
function setRead() {
  const read = mails.value.some(mail => targets.value.includes(mail.id) && !mail.read)
  apply(mail => { mail.read = read })
  notify(read ? '已標記為已讀' : '已標記為未讀')
}
function setStar() {
  const starred = mails.value.some(mail => targets.value.includes(mail.id) && !mail.starred)
  apply(mail => { mail.starred = starred })
}
function moveTrash() {
  apply(mail => { mail.previousFolder = mail.folder; mail.folder = 'trash' })
  checked.value = []
  notify('已移至垃圾桶')
}
function restore() {
  apply(mail => { mail.folder = ['inbox', 'sent', 'drafts', 'spam'].includes(mail.previousFolder) ? mail.previousFolder : 'inbox'; delete mail.previousFolder })
  checked.value = []
  notify('已還原信件')
}
function reset() {
  mails.value = structuredClone(initialMails)
  query.value = ''
  changeFolder('inbox')
  selectedId.value = mails.value[0]?.id
  notify('已恢復原始示範信件')
}
function navigate(offset) {
  const mail = visible.value[selectedIndex.value + offset]
  if (mail) openMail(mail)
}
function compose(mode = 'new') {
  const mail = selected.value
  composeError.value = ''
  if (mode === 'edit' && mail) {
    composer.value = { id: mail.id, to: mail.to, subject: mail.subject, body: mail.body }
  } else {
    composer.value = {
      to: mode === 'reply' && mail ? mail.from.email : '',
      subject: mode === 'new' || !mail ? '' : `${mode === 'reply' ? 'Re' : 'Fwd'}: ${mail.subject}`,
      body: mode === 'new' || !mail ? '' : `\n\n──────── 原始信件 ────────\n${mail.from.name} <${mail.from.email}>\n${mail.body}`,
    }
  }
}
function saveCompose(send = false) {
  const draft = composer.value
  if (send && !/^[^\s@,;]+@[^\s@,;]+\.[^\s@,;]+$/.test(draft.to.trim())) {
    composeError.value = '請輸入有效的收件者 Email。'
    return
  }
  const existing = mails.value.find(mail => mail.id === draft.id)
  const mail = {
    id: draft.id || crypto.randomUUID(), folder: send ? 'sent' : 'drafts',
    from: account, to: draft.to.trim(), subject: draft.subject.trim() || '（無主旨）',
    body: draft.body, date: new Date().toISOString(), read: true, starred: existing?.starred || false,
    attachments: existing?.attachments || [],
  }
  if (existing) Object.assign(existing, mail)
  else mails.value.push(mail)
  composer.value = null
  notify(send ? 'Demo 模擬寄送完成，已加入寄件備份' : '草稿已儲存')
}
</script>

<template>
  <div class="mail-app" :data-theme="theme">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Roundmail 首頁" @click.prevent="changeFolder('inbox')"><span class="brand-mark"><Icon name="mail" /></span><span>round<span class="brand-light">mail</span><small>DEMO</small></span></a>
      <div class="search"><Icon name="search" /><input v-model="query" aria-label="搜尋郵件" placeholder="搜尋郵件" /><button v-if="query" class="icon-button" aria-label="清除搜尋" @click="query = ''"><Icon name="close" /></button><span v-else class="search-hint">{{ folderName }}</span></div>
      <button class="theme-switch" @click="theme = theme === 'classic' ? 'fresh' : 'classic'" :aria-label="`切換至${theme === 'classic' ? '清新' : '經典'}風格`"><span class="theme-swatch"></span>{{ theme === 'classic' ? '經典' : '清新' }}<span class="switch-caption"> · 切換風格</span></button>
      <div class="account"><span class="avatar">夏</span><span>{{ account.email }}<small>個人信箱</small></span></div>
    </header>

    <div class="workspace" :class="{ 'reading-open': mobileReading, 'sidebar-open': sidebarOpen }">
      <aside class="sidebar" aria-label="郵件資料夾">
        <button class="compose-button" :disabled="loading || !!error" @click="compose()"><span><Icon name="plus" /></span>撰寫新郵件</button>
        <div class="folder-heading">我的信箱</div>
        <nav><button v-for="item in folders" :key="item.id" class="folder" :class="{ active: folder === item.id }" :aria-current="folder === item.id ? 'page' : undefined" @click="changeFolder(item.id)"><span class="folder-icon" :style="{ '--folder-color': item.color }"><Icon :name="item.icon" /></span><span>{{ item.label }}</span><b v-if="item.id === 'inbox' && unreadCount">{{ unreadCount }}</b><span v-else class="folder-count">{{ mails.filter(mail => mail.folder === item.id).length }}</span></button></nav>
        <div class="sidebar-bottom"><div class="storage-label"><span>儲存空間</span><span>1.2 GB / 5 GB</span></div><div class="storage-track"><span></span></div><p>純前端展示 · 資料儲存於此瀏覽器</p><button class="reset-button" :disabled="loading || !!error" @click="reset"><Icon name="restore" />重設 Demo</button></div>
      </aside>

      <main class="mail-main">
        <div class="toolbar">
          <button class="icon-button mobile-menu" aria-label="開啟資料夾" @click="sidebarOpen = !sidebarOpen"><Icon name="menu" /></button>
          <label class="select-all" title="全選目前信件"><input type="checkbox" :checked="allChecked" :disabled="!visible.length" @change="toggleAll" /><span>全選</span></label>
          <span class="toolbar-divider"></span>
          <button class="icon-button" title="重新整理清單" aria-label="重新整理清單" :disabled="loading || !!error" @click="query = ''; notify('清單已更新')"><Icon name="refresh" /></button>
          <button class="icon-button" :disabled="!targets.length || !!error" title="切換已讀狀態" aria-label="切換已讀狀態" @click="setRead"><Icon name="mail" /></button>
          <button class="icon-button" :disabled="!targets.length || !!error" title="切換星號" aria-label="切換星號" @click="setStar"><Icon name="star" /></button>
          <span class="toolbar-divider"></span>
          <button v-if="folder !== 'trash'" class="icon-button" :disabled="!targets.length || !!error" title="移至垃圾桶" aria-label="移至垃圾桶" @click="moveTrash"><Icon name="trash" /></button>
          <button v-else class="icon-button" :disabled="!targets.length || !!error" title="還原信件" aria-label="還原信件" @click="restore"><Icon name="restore" /></button>
          <div class="toolbar-summary"><span v-if="checked.length">已選 {{ checked.length }} 封</span><span v-else>{{ folderName }} · {{ visible.length }} 封郵件</span><button class="icon-button" aria-label="上一封" :disabled="selectedIndex <= 0" @click="navigate(-1)"><Icon name="back" /></button><button class="icon-button" aria-label="下一封" :disabled="selectedIndex < 0 || selectedIndex >= visible.length - 1" @click="navigate(1)"><Icon name="next" /></button></div>
        </div>

        <div v-if="loading || error" class="status-panel" role="status"><Icon name="mail" /><h2>{{ loading ? '正在載入信件…' : '信件載入失敗' }}</h2><p>{{ error || '準備您的示範信箱' }}</p><button v-if="error" class="primary-button" @click="load">重新載入</button></div>
        <div v-else class="mail-panes">
          <section class="mail-list" aria-label="信件清單">
            <div class="list-heading"><h1>{{ folderName }}</h1><span>{{ visible.length }} 封信件</span></div>
            <div v-if="!visible.length" class="empty-list"><Icon name="inbox" /><h2>{{ query ? '找不到符合的信件' : '這裡還沒有信件' }}</h2><p>{{ query ? '試試其他關鍵字' : '您的信箱已整理乾淨' }}</p></div>
            <article v-for="mail in visible" :key="mail.id" class="mail-card" :class="{ selected: selectedId === mail.id, unread: !mail.read, checked: checked.includes(mail.id) }">
              <label class="mail-check"><input v-model="checked" type="checkbox" :value="mail.id" :aria-label="`選取 ${mail.subject}`" /></label>
              <button class="mail-open" :aria-label="`閱讀 ${mail.subject}`" @click="openMail(mail)"><div class="mail-subject"><span class="unread-dot" v-if="!mail.read"></span><span>{{ mail.subject }}</span></div><p>{{ mail.body.replace(/\s+/g, ' ') }}</p><div class="mail-meta"><span>{{ ['sent', 'drafts'].includes(folder) ? `收件者：${mail.to || '尚未填寫'}` : mail.from.name }}</span><time :datetime="mail.date">{{ dateLabel(mail.date) }}</time></div></button>
              <Icon v-if="mail.attachments.length" class="attachment-indicator" name="clip" />
              <button class="star-button" :class="{ starred: mail.starred }" :aria-label="mail.starred ? '取消星號' : '加上星號'" :aria-pressed="mail.starred" @click="mail.starred = !mail.starred"><Icon name="star" /></button>
            </article>
            <div v-if="visible.length" class="list-end">以上為所有信件</div>
          </section>

          <section class="reader" aria-label="郵件內容">
            <div v-if="selected" class="reader-sheet">
              <div class="reader-title"><button class="icon-button mobile-back" aria-label="返回信件清單" @click="mobileReading = false"><Icon name="back" /></button><h2>{{ selected.subject }}</h2><button class="reader-star" :class="{ starred: selected.starred }" :aria-label="selected.starred ? '取消星號' : '加上星號'" :aria-pressed="selected.starred" @click="selected.starred = !selected.starred"><Icon name="star" /></button><div class="reader-actions"><button v-if="folder === 'drafts'" @click="compose('edit')"><Icon name="draft" />編輯草稿</button><template v-else><button @click="compose('reply')"><Icon name="reply" />回覆</button><button @click="compose('forward')"><Icon name="forward" />轉寄</button></template></div></div>
              <div class="sender-row"><span class="sender-avatar">{{ selected.from.name.slice(0, 1) }}</span><div><p><strong>{{ selected.from.name }}</strong><span class="sender-email">&lt;{{ selected.from.email }}&gt;</span></p><p class="recipient">收件者：{{ selected.to || '尚未填寫' }}</p></div><time :datetime="selected.date">{{ new Date(selected.date).toLocaleString('zh-TW', { hour12: false }) }}</time></div>
              <div v-if="selected.banner" class="newsletter-banner"><span class="newsletter-mark"><Icon name="mail" /></span><div>{{ selected.banner }}<small>A little inspiration for your everyday.</small></div></div>
              <div class="message-body">{{ selected.body }}</div>
              <div v-if="selected.attachments.length" class="attachments"><h3><Icon name="clip" />{{ selected.attachments.length }} 個附件</h3><a v-for="attachment in selected.attachments" :key="attachment.url" :href="attachment.url" :download="attachment.name"><span class="file-icon"><Icon name="draft" /></span><span><strong>{{ attachment.name }}</strong><small>{{ attachment.size }}</small></span><Icon name="download" /></a></div>
              <div class="reader-footer"><span>— 信件結束 —</span><button v-if="folder !== 'drafts'" @click="compose('reply')"><Icon name="reply" />回覆這封郵件</button></div>
            </div>
            <div v-else class="status-panel"><Icon name="mail" /><h2>給思緒一點空間</h2><p>從左側選取一封郵件，開始閱讀。</p></div>
          </section>
        </div>
      </main>
    </div>

    <div v-if="notice" class="toast" role="status">{{ notice }}</div>
      <dialog v-if="composer" ref="composeDialog" class="composer" aria-labelledby="compose-title" @cancel.prevent="composer = null">
        <header><h2 id="compose-title">{{ composer.id ? '編輯草稿' : '撰寫新郵件' }}</h2><button class="icon-button" aria-label="關閉並放棄此次編輯" @click="composer = null"><Icon name="close" /></button></header>
        <form @submit.prevent="saveCompose(true)">
          <label class="compose-field"><span>收件者</span><input v-model="composer.to" type="text" inputmode="email" placeholder="name@example.com" aria-label="收件者" autofocus /></label>
          <label class="compose-field"><span>主旨</span><input v-model="composer.subject" placeholder="為信件加上主旨" aria-label="主旨" /></label>
          <textarea v-model="composer.body" aria-label="郵件內文" placeholder="寫下您想分享的事…"></textarea>
          <p v-if="composeError" class="compose-error" role="alert">{{ composeError }}</p>
          <footer><button type="submit" class="primary-button"><Icon name="sent" />模擬寄送</button><button type="button" class="secondary-button" @click="saveCompose()">儲存草稿</button><small>Demo 模式，不會寄出真實郵件</small></footer>
        </form>
      </dialog>
  </div>
</template>
