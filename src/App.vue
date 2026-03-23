```vue
<script setup lang="ts">
import { ref, onMounted, computed, watch } from 'vue'

const GAS_URL = 'https://script.google.com/macros/s/AKfycbyKinY8-RUBLwozDnzCQAMo90cX6gVdRREWKV1Yjp3-gvzca1PNk-UE3CM5JDQA4jeGzw/exec'

const activeTab = ref<'VAN_DE_KY_THUAT' | 'PHAT_CODE' | 'XEM_TOAN_BO_CODE'>('VAN_DE_KY_THUAT')

const vanDe = ref<any[]>([])
const phatCode = ref<any[]>([])
const loading = ref(false)
const error = ref<string | null>(null)

const selectedTopic = ref('')
const selectedTopicDetail = ref('')
const editableLines = ref<string[]>([])
const copiedAll = ref(false)
const copiedLineIdx = ref<number | null>(null)

const selectedEvent = ref('')
const copiedPhatCode = ref(false)

const uniqueEvents = computed(() => {
  const events = phatCode.value.map(row => row.EVENT).filter(Boolean)
  return [...new Set(events)]
})

const remainingCount = computed(() => {
  if (!selectedEvent.value) return 0
  return phatCode.value.filter(row => 
    row.EVENT === selectedEvent.value && 
    String(row.isDone).toLowerCase() !== 'true' && row.isDone !== true &&
    String(row.isError).toLowerCase() !== 'true' && row.isError !== true
  ).length
})

const currentCodeRow = computed(() => {
  if (!selectedEvent.value) return null
  return phatCode.value.find(row => 
    row.EVENT === selectedEvent.value && 
    String(row.isDone).toLowerCase() !== 'true' && row.isDone !== true &&
    String(row.isError).toLowerCase() !== 'true' && row.isError !== true
  ) || null
})

const editableReplyForm = ref('')

watch(currentCodeRow, (row) => {
  if (row) {
    editableReplyForm.value = String(row.REPLY_FORM || '').replace(/\\n/g, '\n').replace(/<br\s*\/?>/gi, '\n')
  } else {
    editableReplyForm.value = ''
  }
}, { immediate: true })

watch(editableReplyForm, (val) => {
  if (currentCodeRow.value) {
    currentCodeRow.value.REPLY_FORM = val
  }
})

async function markCode(row: any, status: 'done' | 'error') {
  if (status === 'done') {
    row.isDone = true
  } else {
    row.isError = true
  }
  
  const payload = status === 'done'
    ? { action: 'markDone', code: row.CODE, linkFB: row['LinkFB'], tnv: row['TNV'] }
    : { action: 'markError', code: row.CODE, replyForm: row.REPLY_FORM, linkFB: row['LinkFB'], tnv: row['TNV'] }

  fetch(GAS_URL, {
    method: 'POST',
    body: JSON.stringify(payload),
    headers: { 'Content-Type': 'text/plain;charset=utf-8' }
  }).catch((err) => console.error('Lỗi API Google Sheets:', err))
}

const searchCodeInput = ref('')
const searchResult = ref<any>(null)
const showSearchModal = ref(false)

function handleSearch() {
  const query = searchCodeInput.value.trim()
  if (!query) return
  
  const found = phatCode.value.find(row => String(row.CODE).toLowerCase() === query.toLowerCase())
  if (found) {
    searchResult.value = found
    showSearchModal.value = true
  } else {
    alert('❌ Không tìm thấy mã Code: ' + query)
  }
}

function handleDbSearch() {
  const query = dbSearchCode.value.trim()
  if (!query) return
  
  const found = phatCode.value.find(row => String(row.CODE).toLowerCase() === query.toLowerCase())
  if (found) {
    searchResult.value = found
    showSearchModal.value = true
  } else {
    alert('❌ Không tìm thấy mã Code (' + query + ') trong kho tổng.')
  }
}

function openMobileModal(row: any) {
  searchResult.value = row
  showSearchModal.value = true
}

function closeSearchModal() {
  showSearchModal.value = false
  searchResult.value = null
}

const copiedCodeOnly = ref(false)

function copyJustCode() {
  const row = currentCodeRow.value
  if (!row || !row.CODE) return
  
  navigator.clipboard.writeText(row.CODE)
    .then(() => {
      copiedCodeOnly.value = true
      setTimeout(() => copiedCodeOnly.value = false, 2000)
    })
    .catch((err) => console.error('Lỗi copy:', err))
}

function copyPhatCodeAction() {
  const row = currentCodeRow.value
  if (!row) return
  
  let replyPart = String(row.REPLY_FORM || '').replace(/\\n/g, '\n').replace(/<br\s*\/?>/gi, '\n')
  let fullText = row.CODE
  if (replyPart.trim() && replyPart !== 'undefined') {
    fullText = `${replyPart}\n${row.CODE}`
  }
  
  navigator.clipboard.writeText(fullText)
    .then(() => {
      copiedPhatCode.value = true
      setTimeout(() => copiedPhatCode.value = false, 2000)
    })
    .catch((err) => console.error('Lỗi copy:', err))
}

const uniqueTopics = computed(() => {
  const topics = vanDe.value.map(row => row.TOPIC).filter(Boolean)
  return [...new Set(topics)]
})

const availableDetails = computed(() => {
  if (!selectedTopic.value) return []
  if (selectedTopic.value === 'TẤT CẢ') {
    const details = vanDe.value.map(row => row.TOPIC_DETAILS).filter(d => d)
    return [...new Set(details)]
  }
  const details = vanDe.value
    .filter(row => row.TOPIC === selectedTopic.value)
    .map(row => row.TOPIC_DETAILS)
    .filter(d => d)
  return [...new Set(details)]
})

const selectedRow = computed(() => {
  if (!selectedTopic.value || !selectedTopicDetail.value) return null
  if (selectedTopic.value === 'TẤT CẢ') {
    return vanDe.value.find(row => row.TOPIC_DETAILS === selectedTopicDetail.value)
  }
  return vanDe.value.find(row => row.TOPIC === selectedTopic.value && row.TOPIC_DETAILS === selectedTopicDetail.value)
})

const selectedContent = computed(() => {
  const row = selectedRow.value
  if (!row || !row.REPLY_CONTENT) return ''
  
  // Xử lý các dạng ngắt dòng (newline) từ Google Sheets/Database
  let content = String(row.REPLY_CONTENT)
  content = content.replace(/\\n/g, '\n') // Dịch \n text thành dấu xuống dòng thật
  content = content.replace(/<br\s*\/?>/gi, '\n') // Dịch thẻ <br> thành dấu xuống dòng
  
  return content
})

watch(selectedContent, (newVal) => {
  editableLines.value = newVal.split('\n')
})

const fullContent = computed(() => {
  return editableLines.value.join('\n')
})

function copySingleLine(lineText: string, idx: number) {
  if (!lineText.trim()) return
  navigator.clipboard.writeText(lineText)
    .then(() => {
      copiedLineIdx.value = idx
      setTimeout(() => {
        if (copiedLineIdx.value === idx) copiedLineIdx.value = null
      }, 2000)
    })
    .catch((err) => console.error('Lỗi copy:', err))
}

function copyContent() {
  if (fullContent.value.trim()) {
    navigator.clipboard.writeText(fullContent.value)
      .then(() => {
        copiedAll.value = true
        setTimeout(() => {
          copiedAll.value = false
        }, 2000)
      })
      .catch((err) => console.error('Lỗi copy:', err))
  }
}

function addNewLine() {
  editableLines.value.push('')
}

function removeLine(idx: number) {
  editableLines.value.splice(idx, 1)
}

const vanDeColumns = ['STT', 'TOPIC', 'TOPIC_DETAILS', 'REPLY_CONTENT']
const phatCodeColumns = ['STT', 'CODE', 'DATE', 'isDone', 'isError', 'usingTime', 'REPLY_FORM', 'EVENT', 'LinkFB', 'TNV']
const dbTableColumns = ['STT', 'EVENT', 'CODE', 'DATE', 'isDone', 'isError', 'usingTime', 'LinkFB', 'TNV', 'REPLY_FORM']

const dbSearchCode = ref('')
const dbSelectedEvent = ref('')
const dbSelectedStatus = ref('')
const dbSortOrder = ref<'newest' | 'oldest'>('newest')

function parseVietnameseDateStr(str: any) {
  if (!str) return 0
  if (String(str).includes('T')) return new Date(str).getTime()
  const parts = String(str).split(' ')
  if (parts.length === 2 && parts[1].includes('/')) {
    const timePts = parts[0].split(':')
    const datePts = parts[1].split('/')
    if (datePts.length === 3) {
      return new Date(Number(datePts[2]), Number(datePts[1]) - 1, Number(datePts[0]), Number(timePts[0]||0), Number(timePts[1]||0), Number(timePts[2]||0)).getTime()
    }
  }
  return new Date(str).getTime() || 0
}

const currentPage = ref(1)

watch([dbSearchCode, dbSelectedEvent, dbSelectedStatus, dbSortOrder], () => {
  currentPage.value = 1
})

const filteredDB = computed(() => {
  let result = [...phatCode.value]
  
  if (dbSelectedEvent.value) {
    result = result.filter(row => row.EVENT === dbSelectedEvent.value)
  }

  if (dbSelectedStatus.value) {
    if (dbSelectedStatus.value === 'done') {
      result = result.filter(row => String(row.isDone).toLowerCase() === 'true')
    } else if (dbSelectedStatus.value === 'error') {
      result = result.filter(row => String(row.isError).toLowerCase() === 'true')
    } else if (dbSelectedStatus.value === 'empty') {
      result = result.filter(row => String(row.isDone).toLowerCase() !== 'true' && String(row.isError).toLowerCase() !== 'true')
    }
  }
  
  const query = dbSearchCode.value.trim().toLowerCase()
  if (query) {
    result = result.filter(row => String(row.CODE).toLowerCase().includes(query))
  }
  
  result.sort((a, b) => {
    const tA = parseVietnameseDateStr(a.usingTime)
    const tB = parseVietnameseDateStr(b.usingTime)
    return dbSortOrder.value === 'newest' ? tB - tA : tA - tB
  })
  
  return result
})

const totalPages = computed(() => Math.ceil(filteredDB.value.length / 15))

const paginatedDB = computed(() => {
  const start = (currentPage.value - 1) * 15
  const end = start + 15
  return filteredDB.value.slice(start, end)
})

const visiblePages = computed(() => {
  const maxPages = 5
  let start = Math.max(1, currentPage.value - 2)
  let end = Math.min(totalPages.value, start + maxPages - 1)
  if (end - start + 1 < maxPages) {
    start = Math.max(1, end - maxPages + 1)
  }
  const pages = []
  for (let i = start; i <= end; i++) {
    pages.push(i)
  }
  return pages
})

async function fetchSheet(sheetName: string) {
  const res = await fetch(`${GAS_URL}?sheet=${sheetName}`)
  const json = await res.json()
  if (json.error) throw new Error(json.error)
  return json.rows
}

async function loadAll() {
  loading.value = true
  error.value = null
  try {
    const [v, p] = await Promise.all([
      fetchSheet('VAN_DE_KY_THUAT'),
      fetchSheet('PHAT_CODE'),
    ])
    vanDe.value = v
    phatCode.value = p
  } catch (e: any) {
    error.value = e.message
  } finally {
    loading.value = false
  }
}

onMounted(loadAll)

// Lints cleaned

const labelMap: Record<string, string> = {
  'EVENT': 'SỰ KIỆN (EVENT)',
  'REPLY_FORM': 'MẪU TRẢ LỜI',
  'CODE': 'CODE',
  'DATE': 'HẠN SỬ DỤNG',
  'usingTime': 'THỜI GIAN PHÁT CODE',
  'TNV': 'TNV',
  'LinkFB': 'LINK FB KHÁCH HÀNG',
  'isDone': 'TRẠNG THÁI HOÀN THÀNH',
  'isError': 'TRẠNG THÁI BÁO LỖI'
}

function formatLabel(col: string) {
  return labelMap[col] || col
}

function shouldShowRow(col: string, val: any) {
  if (col === 'STT') return false
  if (col === 'isDone' || col === 'isError') {
    return String(val).toLowerCase() === 'true'
  }
  if (val === null || val === undefined) return false
  return String(val).trim() !== ''
}

function formatData(col: string, value: any) {
  if (!value) return ''
  if (col === 'DATE' || col === 'usingTime') {
    const d = new Date(value)
    if (!isNaN(d.getTime())) {
      const hh = String(d.getHours()).padStart(2, '0')
      const mm = String(d.getMinutes()).padStart(2, '0')
      const ss = String(d.getSeconds()).padStart(2, '0')
      const DD = String(d.getDate()).padStart(2, '0')
      const MM = String(d.getMonth() + 1).padStart(2, '0')
      const YYYY = d.getFullYear()
      return `${hh}:${mm}:${ss} ${DD}/${MM}/${YYYY}`
    }
  }
  if (typeof value === 'string') {
    return value.replace(/\\n/g, '\n').replace(/<br\s*\/?>/gi, '\n')
  }
  return value
}
</script>

<template>
  <div class="app-container">
    <!-- Tabs -->
    <div class="header-controls">
      <div class="tabs-wrapper">
        <button
          v-for="tab in (['VAN_DE_KY_THUAT', 'PHAT_CODE', 'XEM_TOAN_BO_CODE'] as const)"
          :key="tab"
          @click="activeTab = tab"
          :class="['tab-btn', { active: activeTab === tab }]"
        >
          <span class="tab-icon" v-if="tab === 'VAN_DE_KY_THUAT'">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M14.7 6.3a1 1 0 0 0 0 1.4l1.6 1.6a1 1 0 0 0 1.4 0l3.77-3.77a6 6 0 0 1-7.94 7.94l-6.91 6.91a2.12 2.12 0 0 1-3-3l6.91-6.91a6 6 0 0 1 7.94-7.94l-3.76 3.76z"></path></svg>
          </span>
          <span class="tab-icon" v-else-if="tab === 'PHAT_CODE'">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 12 20 22 4 22 4 12"></polyline><rect x="2" y="7" width="20" height="5"></rect><line x1="12" y1="22" x2="12" y2="7"></line><path d="M12 7H7.5a2.5 2.5 0 0 1 0-5C11 2 12 7 12 7z"></path><path d="M12 7h4.5a2.5 2.5 0 0 0 0-5C13 2 12 7 12 7z"></path></svg>
          </span>
          <span class="tab-icon" v-else>
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect><line x1="3" y1="9" x2="21" y2="9"></line><line x1="9" y1="21" x2="9" y2="9"></line></svg>
          </span>
          <span class="tab-text">{{ tab === 'VAN_DE_KY_THUAT' ? 'VẤN ĐỀ KỸ THUẬT' : (tab === 'PHAT_CODE' ? 'PHÁT CODE' : 'QUẢN LÝ KHO CODE') }}</span>
        </button>
      </div>

      <button @click="loadAll" class="refresh-btn">
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 4 23 10 17 10"></polyline><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"></path></svg>
        LÀM MỚI DỮ LIỆU
      </button>
    </div>

    <!-- States -->
    <div v-if="loading" class="status-msg loading">⏳ Đang tải dữ liệu từ hệ thống...</div>
    <div v-else-if="error" class="status-msg error">❌ Lỗi: {{ error }}</div>

    <!-- Content Area: Form or Table -->
    <div v-else>
      <!-- Form View for VAN_DE_KY_THUAT -->
      <div v-if="activeTab === 'VAN_DE_KY_THUAT'" class="form-card">
        <div class="view-header">
          <h2><span>🛠️</span> MẪU VẤN ĐỀ KỸ THUẬT</h2>
          <p>Thu thập thông tin xử lý các sự cố hệ thống và khiếu nại phát sinh</p>
        </div>

        <div class="form-group">
          <label class="form-label">1. CHỦ ĐỀ CHÍNH</label>
          <select v-model="selectedTopic" @change="selectedTopicDetail = ''" class="form-select">
            <option value="" disabled>-- Bấm để chọn chủ đề --</option>
            <option value="TẤT CẢ"> TẤT CẢ</option>
            <option v-for="topic in uniqueTopics" :key="topic" :value="topic">{{ topic }}</option>
          </select>
        </div>
        
        <div class="form-group">
          <label class="form-label">2. CHI TIẾT VẤN ĐỀ</label>
          <select v-model="selectedTopicDetail" class="form-select" :disabled="!selectedTopic">
            <option value="" disabled>-- Bấm để chọn chi tiết --</option>
            <option v-for="detail in availableDetails" :key="detail" :value="detail">{{ detail }}</option>
          </select>
        </div>

        <div class="form-group">
          <label class="form-label">3. NỘI DUNG PHẢN HỒI</label>
          <div class="lines-editor">
            <div v-for="(_, idx) in editableLines" :key="idx" class="line-wrapper">
              <textarea 
                v-model="editableLines[idx]" 
                class="line-textarea"
                placeholder="Nhập nội dung..."
              ></textarea>
              <div class="line-actions">
                <button 
                  @click="copySingleLine(editableLines[idx], idx)" 
                  :class="['action-btn copy', { copied: copiedLineIdx === idx }]"
                  :title="copiedLineIdx === idx ? 'Đã copy' : 'Copy dòng này'"
                >
                  <svg v-if="copiedLineIdx === idx" xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#10b981" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"></polyline></svg>
                  <svg v-else xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path></svg>
                </button>
                <button v-if="editableLines.length > 1" @click="removeLine(idx)" class="action-btn remove" title="Xóa dòng này">
                  <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="3 6 5 6 21 6"></polyline><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path></svg>
                </button>
              </div>
            </div>
            
            <button @click="addNewLine" class="add-line-btn">
              + Thêm dòng trống
            </button>
          </div>
        </div>

        <button @click="copyContent" :disabled="!fullContent.trim()" :class="['copy-btn', { copied: copiedAll }]">
          <span v-if="copiedAll" style="display: flex; gap: 8px; align-items: center; justify-content: center;">
            <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"></polyline></svg>
            ĐÃ SAO CHÉP THÀNH CÔNG
          </span>
          <span v-else style="display: flex; gap: 8px; align-items: center; justify-content: center;">
            <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path></svg>
            SAO CHÉP TẤT CẢ NỘI DUNG
          </span>
        </button>
      </div>
      
      <!-- Form View for PHAT_CODE -->
      <div v-else-if="activeTab === 'PHAT_CODE'" class="form-card">
        <div class="view-header">
          <h2><span>🎁</span> PHÁT CODE</h2>
        </div>

        <!-- SEARCH SECTION VIP COMPACT -->
        <div class="search-vip-container compact">
          <div class="search-vip-box compact-box">
            <div class="search-label-inline">
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#3b82f6" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line></svg>
              <span>TRA CỨU MÃ CODE</span>
            </div>
            <input type="text" v-model="searchCodeInput" @keyup.enter="handleSearch" class="search-vip-input compact-input" placeholder="Gõ mã code vào đây..." />
            <button @click="handleSearch" class="search-vip-btn compact-btn">
              TÌM KIẾM
            </button>
          </div>
        </div>

        <div class="form-group">
          <label class="form-label">1. SỰ KIỆN (EVENT)</label>
          <div style="display: flex; gap: 12px; align-items: stretch;">
            <select v-model="selectedEvent" class="form-select" style="flex: 1;">
              <option value="" disabled>-- Chọn sự kiện --</option>
              <option v-for="evt in uniqueEvents" :key="evt" :value="evt">{{ evt }}</option>
            </select>
            <div v-if="selectedEvent" style="display: flex; align-items: center; padding: 0 24px; background: #eff6ff; border-radius: 12px; color: #2563eb; font-weight: 800; font-size: 16px; border: 2px solid #bfdbfe; white-space: nowrap;">
              Còn lại: {{ remainingCount }}
            </div>
          </div>
        </div>

        <div v-if="!currentCodeRow && selectedEvent" style="padding: 20px; text-align: center; color: #ef4444; font-weight: bold; background: #fee2e2; border-radius: 8px; font-size: 16px;">
          ĐÃ SỬ DỤNG HẾT CODE CHO SỰ KIỆN NÀY
        </div>

        <template v-if="currentCodeRow">
          <div class="form-group">
            <label class="form-label">2. MẪU TRẢ LỜI</label>
            <textarea v-model="editableReplyForm" class="form-textarea" style="min-height: 120px;"></textarea>
          </div>
          
          <div class="form-group">
            <label class="form-label">3. CODE</label>
            <div style="position: relative;">
              <input type="text" :value="currentCodeRow.CODE" readonly class="form-select" style="font-weight: 800; color: #ef4444; text-align: center; font-size: 24px; letter-spacing: 2px; padding: 20px; padding-right: 60px;" />
              <button 
                @click="copyJustCode" 
                :class="['action-btn copy', { copied: copiedCodeOnly }]"
                style="position: absolute; right: 12px; top: 0; bottom: 0; margin: auto; width: 44px; height: 44px;"
                :title="copiedCodeOnly ? 'Đã copy' : 'Copy riêng mã Code này'"
              >
                <svg v-if="copiedCodeOnly" xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#10b981" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"></polyline></svg>
                <svg v-else xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path></svg>
              </button>
            </div>
          </div>

          <div style="display: flex; gap: 16px; margin-top: 4px; margin-bottom: 12px; background: #f8fafc; padding: 16px 20px; border-radius: 12px; border: 1px dashed #cbd5e1;">
            <div style="flex: 1; display: flex; flex-direction: column; gap: 6px;">
              <span style="font-weight: 700; font-size: 14px; color: #64748b; letter-spacing: 0.05em;">4. HẠN SỬ DỤNG</span>
              <span style="font-size: 16px; font-weight: 600; color: #1e293b;">{{ formatData('DATE', currentCodeRow.DATE) || '---' }}</span>
            </div>

            <div style="width: 1px; background: #cbd5e1;"></div>

            <div style="flex: 1; display: flex; flex-direction: column; gap: 6px;">
              <span style="font-weight: 700; font-size: 14px; color: #64748b; letter-spacing: 0.05em;">5. THỜI GIAN PHÁT CODE</span>
              <span style="font-size: 16px; font-weight: 600; color: #1e293b;">{{ formatData('usingTime', currentCodeRow.usingTime) || '---' }}</span>
            </div>
          </div>

          <div style="display: flex; gap: 16px; margin-bottom: 16px;">
            <div class="form-group" style="flex: 1;">
              <label class="form-label" style="display: flex; align-items: center; gap: 6px;">
                <span>👤</span> TNV
              </label>
              <input type="text" v-model="currentCodeRow['TNV']" class="form-select" placeholder="Mã điều phối..." style="background: #fff; font-weight: 600; font-size: 15px;" />
            </div>
            
            <div class="form-group" style="flex: 2;">
              <label class="form-label" style="display: flex; align-items: center; gap: 6px;">
                <span>🔗</span> LINK FB KHÁCH HÀNG
              </label>
              <input type="text" v-model="currentCodeRow['LinkFB']" class="form-select" placeholder="Dán link Facebook..." style="background: #fff; font-weight: 600; font-size: 15px;" />
            </div>
          </div>

          <button @click="copyPhatCodeAction" :class="['copy-btn', { copied: copiedPhatCode }]" style="margin-bottom: 20px;">
            <span v-if="copiedPhatCode" style="display: flex; gap: 8px; align-items: center; justify-content: center;">
              <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"></polyline></svg>
              ĐÃ SAO CHÉP THÀNH CÔNG
            </span>
            <span v-else style="display: flex; gap: 8px; align-items: center; justify-content: center;">
              <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path></svg>
              SAO CHÉP (MẪU TRẢ LỜI + CODE)
            </span>
          </button>

          <div class="action-btns-row">
            <button @click="markCode(currentCodeRow, 'error')" class="status-btn error-btn" style="flex: 1;">
              <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"></path><line x1="12" y1="9" x2="12" y2="13"></line><line x1="12" y1="17" x2="12.01" y2="17"></line></svg>
              BÁO LỖI
            </button>
            <button @click="markCode(currentCodeRow, 'done')" class="status-btn done-btn" style="flex: 2;">
              <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"></polyline></svg>
              HOÀN THÀNH
            </button>
          </div>
        </template>
      </div>

      <!-- Table View for XEM TOÀN BỘ CODE -->
      <div v-else-if="activeTab === 'XEM_TOAN_BO_CODE'" class="table-container">

        <div class="view-header table-view-header">
          <h2><span>🗄️</span> KHO CODE TỔNG QUÁT</h2>
          <p>Theo dõi và nắm bắt trực quan toàn bộ trạng thái vòng đời của mã Code</p>
        </div>
        
        <!-- BỘ LỌC VÀ TÌM KIẾM CHO DB DẠNG HÀNG NGANG (1 ROW) -->
        <div class="db-filter-row">
          
          <div class="search-vip-container compact" style="margin-bottom: 0; flex: 2; padding: 6px 12px; height: auto;">
            <div class="search-vip-box compact-box" style="margin-top: 0;">
              <div class="search-label-inline" style="margin-right: 8px;">
                <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#3b82f6" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line></svg>
                <span style="font-size: 13px;">MÃ CODE:</span>
              </div>
              <input type="text" v-model="dbSearchCode" @keyup.enter="handleDbSearch" class="search-vip-input compact-input" style="height: 42px; padding: 8px 12px; font-size: 14px; border: none; box-shadow: none;" placeholder="Gõ Code vào đây..." />
              <button @click="handleDbSearch" class="search-vip-btn compact-btn">
                Search
              </button>
            </div>
          </div>

          <div style="flex: 1; display: flex; align-items: center; gap: 10px;">
            <span style="font-weight: 700; font-size: 13px; color: #64748b; white-space: nowrap;">SỰ KIỆN:</span>
            <select v-model="dbSelectedEvent" class="form-select" style="margin: 0; padding: 8px 12px; height: 42px; font-size: 14px; font-weight: 600;">
              <option value="">-- Tất Cả Kho --</option>
              <option v-for="evt in uniqueEvents" :key="evt" :value="evt">
                {{ evt }}
              </option>
            </select>
          </div>

          <div style="flex: 1; display: flex; align-items: center; gap: 10px;">
            <span style="font-weight: 700; font-size: 13px; color: #64748b; white-space: nowrap;">TRẠNG THÁI:</span>
            <select v-model="dbSelectedStatus" class="form-select" style="margin: 0; padding: 8px 12px; height: 42px; font-size: 14px; font-weight: 600;">
              <option value="">-- Tất Cả --</option>
              <option value="done">✅ Hoàn Thành</option>
              <option value="error">⚠️ Báo Lỗi</option>
              <option value="empty">⏳ Chưa Dùng</option>
            </select>
          </div>

          <div style="flex: 1; display: flex; align-items: center; gap: 10px;">
            <span style="font-weight: 700; font-size: 13px; color: #64748b; white-space: nowrap;">THỜI GIAN:</span>
            <select v-model="dbSortOrder" class="form-select" style="margin: 0; padding: 8px 12px; height: 42px; font-size: 14px; font-weight: 600;">
              <option value="newest">Mới Nhất</option>
              <option value="oldest">Cũ Nhất</option>
            </select>
          </div>
        </div>

        <table class="data-table desktop-table">
          <thead>
            <tr>
              <th v-for="col in dbTableColumns" :key="col" style="white-space: nowrap;">
                {{ formatLabel(col) }}
              </th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, i) in paginatedDB" :key="i">
              <td v-for="col in dbTableColumns" :key="col" :style="col === 'STT' ? 'text-align: center; font-weight: bold; color: #64748b;' : ''" :data-label="formatLabel(col)">
                <span v-if="col === 'STT'">
                  {{ (currentPage - 1) * 15 + i + 1 }}
                </span>
                <span v-else-if="col === 'isDone'">
                  <span v-if="String(row[col]).toLowerCase() === 'true'" style="color: #10b981; font-weight: 800; background: #d1fae5; padding: 4px 10px; border-radius: 8px;">DONE</span>
                </span>
                <span v-else-if="col === 'isError'">
                  <span v-if="String(row[col]).toLowerCase() === 'true'" style="color: #ef4444; font-weight: 800; background: #fee2e2; padding: 4px 10px; border-radius: 8px;">ERROR</span>
                </span>
                <span v-else-if="col === 'LinkFB'">
                  <a v-if="row[col]" :href="row[col]" target="_blank" style="color: #3b82f6; text-decoration: underline; font-weight: 600;">
                    Thăm KH 🔗
                  </a>
                  <span v-else>---</span>
                </span>
                <span v-else-if="col === 'usingTime'" style="color: #047857; font-weight: 700;">
                  {{ formatData(col, row[col]) }}
                </span>
                <span v-else-if="col === 'CODE'" style="color: #1d4ed8; font-weight: 700;">
                  {{ formatData(col, row[col]) }}
                </span>
                <span v-else>{{ formatData(col, row[col]) }}</span>
              </td>
            </tr>
            <tr v-if="paginatedDB.length === 0">
              <td :colspan="dbTableColumns.length" style="text-align: center; padding: 40px; color: #94a3b8; font-size: 16px;">
                📭 Không tìm thấy dữ liệu CODE nào phù hợp (Rỗng)
              </td>
            </tr>
          </tbody>
        </table>

        <!-- NATIVE MOBILE CARDS UI -->
        <div class="mobile-cards-list">
          <div v-if="paginatedDB.length === 0" style="text-align: center; padding: 40px; color: #94a3b8; font-size: 16px;">
            📭 Không tìm thấy kết quả nào
          </div>
          <div 
             v-for="(row, i) in paginatedDB" 
             :key="'mob-'+i" 
             class="mob-code-card"
             @click="openMobileModal(row)"
          >
            <!-- 1. Trạng thái -->
            <div class="mob-status">
               <span v-if="String(row.isDone).toLowerCase() === 'true'" class="mob-pill done">✅ HOÀN THÀNH</span>
               <span v-else-if="String(row.isError).toLowerCase() === 'true'" class="mob-pill error">⚠️ BÁO LỖI</span>
               <span v-else class="mob-pill pending">⏳ CHƯA SỬ DỤNG</span>
            </div>
            
            <!-- 2. Sự kiện -->
            <div class="mob-event">
               <span style="font-size: 12px; color: #64748b; font-weight: 600; display: block; margin-bottom: 2px;">SỰ KIỆN:</span>
               {{ row.EVENT || '---' }}
            </div>
            
            <!-- 3. Code (Giant Pill) -->
            <div class="mob-code">
               {{ row.CODE || '---' }}
            </div>
          </div>
        </div>

        <!-- BỘ ĐIỀU KHIỂN PHÂN TRANG -->
        <div v-if="totalPages > 1" class="pagination-footer">
          <span style="font-size: 14px; color: #64748b; font-weight: 600;">
            Trang {{ currentPage }} / {{ totalPages }} (Tổng {{ filteredDB.length }} mã)
          </span>
          <div style="display: flex; gap: 8px;">
            <button 
              @click="currentPage > 1 && currentPage--" 
              :disabled="currentPage === 1"
              class="page-btn"
            >
              Trang Trước
            </button>
            
            <div style="display: flex; gap: 4px;">
               <button 
                 v-for="p in visiblePages" :key="p"
                 @click="currentPage = p"
                 :class="['page-num-btn', { active: currentPage === p }]"
               >
                 {{ p }}
               </button>
            </div>

            <button 
              @click="currentPage < totalPages && currentPage++" 
              :disabled="currentPage === totalPages"
              class="page-btn"
            >
              Trang Sau
            </button>
          </div>
        </div>

      </div>
    </div>

    <!-- Tra cứu Modal -->
    <div v-if="showSearchModal && searchResult" class="modal-overlay" @click.self="closeSearchModal">
      <div class="modal-card">
        <div class="modal-header">
          <h3>Chi Tiết Code: <span style="color: #ef4444; letter-spacing: 1px;">{{ searchResult.CODE }}</span></h3>
          <button @click="closeSearchModal" class="close-btn">✕</button>
        </div>
        <div class="modal-body">
          <table class="detail-table">
            <tbody>
              <template v-for="col in phatCodeColumns" :key="col">
                <tr v-if="shouldShowRow(col, searchResult[col])">
                  <td class="col-label">{{ formatLabel(col) }}</td>
                  <td class="col-value">
                    <span v-if="col === 'isDone'">
                      <span v-if="String(searchResult[col]).toLowerCase() === 'true'" style="color: #10b981; font-weight: 800; background: #d1fae5; padding: 4px 12px; border-radius: 8px;">DONE</span>
                    </span>
                    <span v-else-if="col === 'isError'">
                      <span v-if="String(searchResult[col]).toLowerCase() === 'true'" style="color: #ef4444; font-weight: 800; background: #fee2e2; padding: 4px 12px; border-radius: 8px;">ERROR</span>
                    </span>
                    <span v-else-if="col === 'LinkFB'">
                      <a v-if="searchResult[col]" :href="searchResult[col]" target="_blank" style="color: #3b82f6; text-decoration: underline; font-weight: 600;">
                        {{ searchResult[col] }}
                      </a>
                      <span v-else>---</span>
                    </span>
                    <span v-else-if="col === 'usingTime'" style="color: #047857; font-weight: 700;">
                      {{ formatData(col, searchResult[col]) || '---' }}
                    </span>
                    <span v-else-if="col === 'CODE'" style="color: #1d4ed8; font-weight: 700;">
                      {{ formatData(col, searchResult[col]) || '---' }}
                    </span>
                    <span v-else>{{ formatData(col, searchResult[col]) || '---' }}</span>
                  </td>
                </tr>
              </template>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
/* Override default Vite/Vue constrained canvas */
#app {
  max-width: 100% !important;
  width: 100%;
  padding: 0 !important;
}
body {
  margin: 0;
  padding: 0;
  width: 100%;
}
</style>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

.app-container {
  padding: 30px 48px;
  max-width: 100%;
  margin: 0 auto;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  min-height: 100vh;
  box-sizing: border-box;
}

/* HEADER CONTROLS (TABS SELECTION) */
.header-controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 32px;
  gap: 16px;
}

.tabs-wrapper {
  display: flex;
  background: #ffffff;
  padding: 8px;
  border-radius: 14px;
  gap: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.tab-btn {
  padding: 14px 28px;
  border-radius: 10px;
  border: none;
  cursor: pointer;
  font-weight: 700;
  font-size: 16px;
  transition: all 0.3s ease;
  background: #f1f5f9;
  color: #64748b;
  box-shadow: 0 2px 4px rgba(0,0,0,0.02);
}

.tab-btn.active {
  background: linear-gradient(135deg, #1e293b, #0f172a);
  color: #fff;
  box-shadow: 0 8px 16px rgba(15, 23, 42, 0.2);
  transform: translateY(-1px);
}

.tab-btn:hover:not(.active) {
  background: #e2e8f0;
  color: #334155;
}

.refresh-btn {
  margin-left: auto;
  padding: 14px 28px;
  border-radius: 10px;
  border: none;
  cursor: pointer;
  font-weight: 700;
  font-size: 15px;
  background: linear-gradient(135deg, #3b82f6, #2563eb);
  color: white;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.25);
}

.refresh-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(37, 99, 235, 0.35);
}

.form-card {
  max-width: 1200px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 28px;
  background: white;
  padding: 40px;
  border-radius: 20px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.06);
  border: 1px solid #f1f5f9;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.form-label {
  font-weight: 700;
  font-size: 15px;
  color: #334155;
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

.form-select, .form-textarea {
  padding: 16px 20px;
  border-radius: 12px;
  border: 2px solid #e2e8f0;
  font-size: 17px;
  background: #f8fafc;
  color: #0f172a;
  outline: none;
  transition: all 0.3s ease;
  font-family: inherit;
  font-weight: 500;
  line-height: 1.6;
  width: 100%;
  box-sizing: border-box;
}

.form-select:hover:not(:disabled), .form-textarea:hover:not([readonly]) {
  border-color: #cbd5e1;
}

.form-select:focus, .form-textarea:focus {
  border-color: #3b82f6;
  background: #fff;
  box-shadow: 0 0 0 4px rgba(59, 130, 246, 0.15);
}

.form-select:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.lines-editor {
  display: flex;
  flex-direction: column;
  gap: 12px;
  background: #f8fafc;
  padding: 16px;
  border-radius: 12px;
  border: 2px solid #e2e8f0;
}

.line-wrapper {
  position: relative;
  display: flex;
  align-items: flex-start;
}

.line-textarea {
  flex: 1;
  padding: 14px;
  padding-right: 80px;
  border-radius: 10px;
  border: 1px solid #cbd5e1;
  outline: none;
  font-family: inherit;
  font-size: 16px;
  resize: vertical;
  min-height: 50px;
  background: #fff;
  color: #1e293b;
  transition: all 0.2s;
  line-height: 1.5;
  box-sizing: border-box;
}

.line-textarea:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.line-actions {
  position: absolute;
  right: 8px;
  top: 8px;
  display: flex;
  gap: 4px;
}

.action-btn {
  width: 34px;
  height: 34px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 8px;
  background: transparent;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
  color: #94a3b8;
}

.action-btn.copy:hover {
  background: #eff6ff;
  color: #3b82f6;
}

.action-btn.copy.copied {
  background: #d1fae5;
  color: #10b981;
  border-color: #a7f3d0;
  transform: scale(1.05);
}

.action-btn.remove:hover {
  background: #fee2e2;
  color: #ef4444;
}

.action-btn:active {
  transform: scale(0.95);
}

.add-line-btn {
  padding: 12px;
  border-radius: 10px;
  border: 2px dashed #cbd5e1;
  background: transparent;
  color: #64748b;
  font-weight: 700;
  font-size: 15px;
  cursor: pointer;
  transition: all 0.2s;
}

.add-line-btn:hover {
  border-color: #94a3b8;
  color: #334155;
  background: #f1f5f9;
}

.copy-btn {
  padding: 18px;
  border-radius: 12px;
  border: none;
  font-weight: 700;
  font-size: 18px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 12px;
  transition: all 0.3s ease;
  margin-top: 10px;
}

.copy-btn:not(:disabled) {
  background: linear-gradient(135deg, #3b82f6, #2563eb);
  color: white;
  cursor: pointer;
  box-shadow: 0 8px 20px rgba(59, 130, 246, 0.25);
}

.copy-btn:not(:disabled):hover {
  transform: translateY(-3px);
  box-shadow: 0 12px 24px rgba(59, 130, 246, 0.35);
  background: linear-gradient(135deg, #2563eb, #1d4ed8);
}

.copy-btn.copied {
  background: linear-gradient(135deg, #10b981, #059669) !important;
  box-shadow: 0 4px 12px rgba(16, 185, 129, 0.4) !important;
  transform: scale(0.98) !important;
}

.copy-btn:disabled {
  background: #e2e8f0;
  color: #94a3b8;
  cursor: not-allowed;
}

.status-btn {
  padding: 18px;
  border-radius: 12px;
  font-weight: 800;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 10px;
  letter-spacing: 0.5px;
}

.done-btn {
  background: linear-gradient(135deg, #10b981, #059669);
  color: white;
  border: none;
  box-shadow: 0 8px 20px rgba(16, 185, 129, 0.25);
}

.done-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 12px 24px rgba(16, 185, 129, 0.35);
  background: linear-gradient(135deg, #059669, #047857);
}

.done-btn:active {
  transform: scale(0.98);
}

.error-btn {
  background: white;
  color: #ef4444;
  border: 2px solid #fecaca;
  box-shadow: 0 4px 12px rgba(239, 68, 68, 0.05);
}

.error-btn:hover {
  background: #fef2f2;
  border-color: #f87171;
  transform: translateY(-2px);
  box-shadow: 0 8px 16px rgba(239, 68, 68, 0.1);
}

.error-btn:active {
  transform: scale(0.98);
}

.table-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
  overflow-x: auto;
  max-width: 100%;
  margin: 0 auto;
}

.data-table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  font-size: 16px;
}

.data-table th {
  background: #f8fafc;
  color: #475569;
  padding: 18px;
  text-align: left;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border-bottom: 2px solid #e2e8f0;
}

.data-table th:first-child {
  border-top-left-radius: 12px;
}

.data-table th:last-child {
  border-top-right-radius: 12px;
}

.data-table td {
  padding: 18px;
  border-bottom: 1px solid #f1f5f9;
  color: #334155;
  white-space: pre-wrap;
  vertical-align: top;
  line-height: 1.6;
}

.data-table tr:hover td {
  background: #f8fafc;
}

.status-msg {
  text-align: center;
  font-size: 20px;
  font-weight: 600;
  padding: 60px;
  border-radius: 20px;
  background: white;
  box-shadow: 0 10px 30px rgba(0,0,0,0.05);
  margin-top: 20px;
}

.loading {
  color: #3b82f6;
}

.error {
  color: #ef4444;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(15, 23, 42, 0.4);
  backdrop-filter: blur(4px);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}

.modal-card {
  background: white;
  width: 90%;
  max-width: 650px;
  border-radius: 20px;
  box-shadow: 0 20px 50px rgba(0,0,0,0.15);
  overflow: hidden;
  animation: modalIn 0.3s ease;
}

@keyframes modalIn {
  from { opacity: 0; transform: translateY(20px) scale(0.95); }
  to { opacity: 1; transform: translateY(0) scale(1); }
}

.modal-header {
  padding: 24px 30px;
  background: #f8fafc;
  border-bottom: 2px solid #e2e8f0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h3 {
  margin: 0;
  font-size: 22px;
  color: #1e293b;
}

.close-btn {
  background: transparent;
  border: none;
  font-size: 24px;
  color: #64748b;
  cursor: pointer;
  transition: all 0.2s;
  padding: 0;
  line-height: 1;
}

.close-btn:hover {
  color: #ef4444;
  transform: scale(1.1);
}

.modal-body {
  padding: 30px;
  max-height: 70vh;
  overflow-y: auto;
}

.detail-table {
  width: 100%;
  border-collapse: collapse;
}

.detail-table td {
  padding: 16px;
  border-bottom: 1px solid #f1f5f9;
}

.col-label {
  font-weight: 700;
  color: #64748b;
  width: 35%;
  background: #f8fafc;
  border-radius: 8px 0 0 8px;
}

.col-value {
  color: #1e293b;
  font-size: 16px;
  background: #fff;
  border-radius: 0 8px 8px 0;
  white-space: pre-wrap;
  word-break: break-word;
}

/* SEARCH VIP STYLES */
.search-vip-container {
  background: linear-gradient(145deg, #ffffff, #f8fafc);
  padding: 24px;
  border-radius: 16px;
  border: 1px solid #e2e8f0;
  box-shadow: 0 8px 20px rgba(0,0,0,0.03);
  margin-bottom: 24px;
  position: relative;
  overflow: hidden;
}

.search-vip-container::before {
  content: '';
  position: absolute;
  top: 0; left: 0;
  width: 4px; height: 100%;
  background: linear-gradient(to bottom, #3b82f6, #8b5cf6);
}

.search-vip-box {
  display: flex;
  gap: 12px;
  margin-top: 12px;
}

.search-input-wrapper {
  flex: 1;
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 16px;
  pointer-events: none;
}

.search-vip-input {
  width: 100%;
  padding: 16px 20px 16px 48px;
  border-radius: 12px;
  border: 2px solid #e2e8f0;
  font-size: 16px;
  font-weight: 600;
  letter-spacing: 0.5px;
  background: #ffffff;
  color: #1e293b;
  outline: none;
  transition: all 0.3s ease;
  box-sizing: border-box;
}

.search-vip-input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 4px rgba(59, 130, 246, 0.15);
}

.search-vip-input::placeholder {
  color: #94a3b8;
  font-weight: 400;
  letter-spacing: normal;
}

.search-vip-btn {
  background: linear-gradient(135deg, #1e293b, #0f172a);
  color: white;
  border: none;
  padding: 0 28px;
  border-radius: 12px;
  font-weight: 700;
  font-size: 15px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 12px rgba(15, 23, 42, 0.2);
  white-space: nowrap;
}

.search-vip-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 16px rgba(15, 23, 42, 0.3);
  background: linear-gradient(135deg, #334155, #1e293b);
}

.search-vip-btn:active {
  transform: scale(0.98);
}

.search-vip-container.compact {
  padding: 12px 16px;
  background: #f8fafc;
  border-radius: 12px;
  border: 1px solid #cbd5e1;
  box-shadow: none;
  margin-bottom: 24px;
}

.search-vip-container.compact::before {
  display: none;
}

.search-vip-box.compact-box {
  margin-top: 0;
  display: flex;
  gap: 12px;
  align-items: center;
}

.search-label-inline {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 700;
  color: #1e293b;
  font-size: 14px;
  white-space: nowrap;
}

.search-vip-input.compact-input {
  padding: 10px 14px;
  font-size: 15px;
  border-radius: 8px;
  flex: 1;
}

.search-vip-btn.compact-btn {
  padding: 0 20px;
  height: 44px;
  border-radius: 8px;
  font-size: 14px;
}

/* PAGINATION STYLES */
.page-btn {
  padding: 8px 16px;
  background: white;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-weight: 600;
  color: #475569;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 13px;
}
.page-btn:not(:disabled):hover {
  background: #f1f5f9;
  color: #1e293b;
  border-color: #94a3b8;
}
.page-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.page-num-btn {
  padding: 8px 12px;
  background: white;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-weight: 600;
  color: #475569;
  cursor: pointer;
  transition: all 0.2s;
  min-width: 36px;
  font-size: 13px;
}
.page-num-btn:hover {
  background: #f1f5f9;
  border-color: #94a3b8;
}
.page-num-btn.active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

/* VIEW HEADER STYLES */
.view-header {
  margin-bottom: 32px;
  padding-bottom: 20px;
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.view-header::before {
  content: '';
  position: absolute;
  bottom: 0; left: 0;
  height: 2px; width: 100%;
  background: #f1f5f9;
}

.view-header::after {
  content: '';
  position: absolute;
  bottom: 0; left: 0;
  height: 2px; width: 64px;
  background: linear-gradient(90deg, #3b82f6, #8b5cf6);
  border-radius: 2px;
}

.view-header h2 {
  margin: 0;
  color: #0f172a;
  font-size: 24px;
  font-weight: 800;
  letter-spacing: -0.5px;
  display: flex;
  align-items: center;
  gap: 14px;
}

.view-header h2 span {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 44px;
  height: 44px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  font-size: 22px !important;
  box-shadow: 0 2px 4px rgba(0,0,0,0.02);
}

.view-header p {
  margin: 0;
  color: #64748b;
  font-size: 14px;
  font-weight: 500;
  padding-left: 58px;
}

.table-view-header {
  padding: 24px 24px 20px 24px;
  margin-bottom: 0;
}
.table-view-header::before {
  left: 24px;
  width: calc(100% - 48px);
}
.table-view-header::after {
  left: 24px;
}

/* RESPONSIVE LAYOUT CLASSES */
.action-btns-row {
  display: flex;
  gap: 16px;
}

.db-filter-row {
  padding: 16px 24px;
  background: #ffffff;
  border-bottom: 2px solid #e2e8f0;
  display: flex;
  align-items: center;
  gap: 24px;
  flex-wrap: nowrap;
}

.pagination-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 24px;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
}

.mobile-cards-list {
  display: none;
}

/* MOBILE RESPONSIVE MEDIA QUERIES */
@media (max-width: 860px) {
  /* Prevent iOS Safari Auto-Zoom on form focus */
  input, select, textarea, .form-input, .form-select, .search-vip-input {
    font-size: 16px !important;
  }

  .app-container {
    padding: 16px 12px;
  }
  
  .header-controls {
    flex-direction: column;
    align-items: stretch;
    gap: 12px;
  }
  
  .tabs-wrapper {
    flex-direction: column;
    gap: 6px;
  }

  .tab-btn, .refresh-btn {
    width: 100%;
    justify-content: center;
  }
  
  .view-header {
    padding: 16px;
    margin-bottom: 16px;
  }
  
  .view-header h2 {
    font-size: 20px;
    flex-wrap: wrap;
  }

  .view-header p {
    padding-left: 0;
    margin-top: 8px;
  }
  
  .table-view-header {
    padding: 16px;
  }
  
  /* FIXED SEARCH BOX MOBILE SQUISH */
  .search-vip-box.compact-box {
    flex-direction: column;
    align-items: stretch;
    padding: 12px !important;
    gap: 8px;
  }
  .search-label-inline {
    margin-right: 0 !important;
    justify-content: center;
  }
  .search-vip-input.compact-input {
    width: 100%;
    box-sizing: border-box;
  }
  .search-vip-btn.compact-btn {
    width: 100%;
  }

  .db-filter-row {
    flex-direction: column;
    align-items: stretch;
    padding: 16px;
    gap: 16px;
  }
  
  .db-filter-row > div, .db-filter-row > .search-vip-container {
    width: 100%;
    flex: none !important;
    box-sizing: border-box;
  }

  .form-card {
    padding: 20px 16px;
  }

  .action-btns-row {
    flex-direction: column;
    gap: 12px;
  }

  .pagination-footer {
    flex-direction: column;
    gap: 16px;
    padding: 16px;
  }

  .pagination-footer > div {
    width: 100%;
    justify-content: space-between;
  }

  .desktop-table {
    display: none !important;
  }
  
  .mobile-cards-list {
    display: flex;
    flex-direction: column;
    gap: 16px;
    padding: 16px;
    background: #f8fafc;
  }
  
  .mob-code-card {
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 16px;
    padding: 16px;
    display: flex;
    flex-direction: column;
    gap: 12px;
    box-shadow: 0 4px 10px rgba(0,0,0,0.04);
    cursor: pointer;
    transition: transform 0.15s ease-out, box-shadow 0.15s;
  }

  .mob-code-card:active {
    transform: scale(0.97);
    box-shadow: 0 2px 4px rgba(0,0,0,0.02);
  }
  
  .mob-status {
    display: flex;
  }
  
  .mob-pill {
    font-size: 12px;
    font-weight: 800;
    padding: 6px 14px;
    border-radius: 20px;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    letter-spacing: 0.5px;
  }
  
  .mob-pill.done { background: #d1fae5; color: #10b981; }
  .mob-pill.error { background: #fee2e2; color: #ef4444; }
  .mob-pill.pending { background: #f1f5f9; color: #64748b; }
  
  .mob-event {
    background: #f8fafc;
    border: 1px solid #f1f5f9;
    padding: 10px 16px;
    border-radius: 10px;
    font-size: 15px;
    font-weight: 700;
    color: #1e293b;
  }
  
  .mob-code {
    background: #eff6ff;
    border: 2px dashed #93c5fd;
    padding: 16px;
    border-radius: 14px;
    font-size: 20px;
    font-weight: 800;
    color: #1d4ed8;
    text-align: center;
    letter-spacing: 1.5px;
  }
}
</style>