<template>
  <div class="layout">
    <aside class="sidebar">
      <div class="brand">智能语音机器人</div>
      <div class="menu-group">
        <div class="menu-title">触达管理</div>
        <div class="menu-item">触达任务</div>
        <div class="menu-item">触达策略</div>
        <div class="menu-item active">通话明细</div>
        <div class="menu-item">任务执行明细</div>
        <div class="menu-item">消息触达明细</div>
      </div>
      <div class="menu-group">
        <div class="menu-item">AI机器人</div>
        <div class="menu-item">甲方管理</div>
        <div class="menu-item">预警中心</div>
        <div class="menu-item">BI报表</div>
      </div>
    </aside>

    <div class="main-area">
      <header class="topbar">
        <div class="breadcrumb">触达管理 / 通话明细</div>
        <div class="user">mengxiyuan@cloudun.ai</div>
      </header>

      <div class="content">
        <div class="page-title">通话明细</div>

        <div class="card">
          <div class="filters">
            <div class="filter-row">
              <label class="filter-label">项目名称</label>
              <input v-model="projectKeyword" class="filter-input" placeholder="请输入" />

              <label class="filter-label">时间</label>
              <input class="filter-input date-input" value="2026-02-20 00:00 至 2026-02-27 23:59" readonly />

              <label class="filter-label">策略</label>
              <input v-model="strategyKeyword" class="filter-input" placeholder="请输入" />

              <label class="filter-label">标签</label>
              <input class="filter-input" placeholder="请选择" />
            </div>

            <div class="filter-row">
              <label class="filter-label">电话号码</label>
              <input v-model="phoneKeyword" class="filter-input" placeholder="请输入" />

              <label class="filter-label">呼叫UID</label>
              <input v-model="uidKeyword" class="filter-input uid-input" placeholder="请输入" />

              <label class="filter-label">轮次</label>
              <input v-model="roundKeyword" class="filter-input round-input" placeholder="请输入" />

              <label class="filter-label">触达方式</label>
              <select v-model="touchModeKeyword" class="filter-input touch-mode-select">
                <option value="">全部</option>
                <option value="机器人">机器人</option>
                <option value="IVR">IVR</option>
              </select>

              <button class="btn btn-primary small-btn" @click="onSearch">查询</button>
              <button class="btn btn-secondary small-btn" @click="onReset">重置</button>
            </div>
          </div>

          <table class="table">
            <thead>
            <tr>
              <th>呼叫UID</th>
              <th>任务名称</th>
              <th>策略名称</th>
              <th>轮次</th>
              <th>触达方式</th>
              <th>用户姓名</th>
              <th>电话</th>
              <th>操作</th>
            </tr>
            </thead>
            <tbody>
            <tr
              v-for="item in filteredCallData"
              :key="item.uid"
            >
              <td class="ellipsis-cell">{{ item.uid }}</td>
              <td>{{ item.taskName }}</td>
              <td class="ellipsis-cell">{{ item.strategy }}</td>
              <td>{{ item.round }}</td>
              <td>{{ item.touchMode || '--' }}</td>
              <td class="ellipsis-cell">{{ item.userName }}</td>
              <td>{{ formatPhone(item.phone) }}</td>
              <td>
                <button class="action-link action-link-detail" @click="openDrawer(item)">详情</button>
                <span class="action-divider"> | </span>
                <button class="action-link action-link-play">播放</button>
              </td>
            </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- 抽屉遮罩 -->
    <div
      class="drawer-mask"
      :class="{ show: showDrawer }"
      @click="closeDrawer"
    ></div>

    <!-- 抽屉 -->
    <div class="drawer" :class="{ show: showDrawer }">
      <div class="drawer-header">
        <div class="drawer-header-left">
          <div class="drawer-title">通话详情</div>
          <div class="call-id-row">
            <span class="call-id-label">呼叫ID</span>
            <span class="call-id-value">{{ currentCallId }}</span>
            <button class="copy-btn" @click="copyCurrentCallId">
              {{ copiedCallId ? '已复制' : '复制' }}
            </button>
          </div>
        </div>
        <div class="drawer-close" @click="closeDrawer">&times;</div>
      </div>

      <div class="drawer-body" v-if="currentRow">
        <div class="detail-left">
          <div class="section audio-section">
            <div class="section-title">通话录音</div>
            <div class="audio-wrapper">
              <audio
                :key="currentAudioUrl"
                controls
                :src="currentAudioUrl"
                preload="metadata"
              >
                您的浏览器不支持 audio 标签
              </audio>
            </div>
          </div>

          <div class="section dialog-section">
            <div class="section-title">对话内容</div>
            <div class="chat-box">
              <div
                v-for="(line, i) in currentDialogLines"
                :key="i"
                class="dialog-item"
              >
                <div
                  v-if="!line.noSpeech"
                  class="dialog-time-row"
                  :class="line.speaker === '用户' ? 'dialog-time-user' : 'dialog-time-agent'"
                >
                  {{ line.time }}
                </div>

                <!-- 左侧：AI -->
                <div
                  v-if="line.speaker !== '用户' && !line.noSpeech"
                  class="dialog-main dialog-main-agent"
                >
                  <div class="dialog-avatar">
                    <span>AI</span>
                  </div>
                  <div class="dialog-bubble dialog-bubble-agent">
                    {{ getAgentLineText(line) }}
                  </div>
                </div>

                <!-- 右侧：用户 -->
                <div
                  v-else-if="line.speaker === '用户' && !line.noSpeech"
                  class="dialog-main dialog-main-user"
                >
                  <div
                    class="dialog-bubble dialog-bubble-user"
                  >
                    {{ getLineText(line) }}
                  </div>
                  <div class="dialog-avatar">
                    <span>用户</span>
                  </div>
                </div>

                <!-- 用户意图：展示在用户回复文字下方 -->
                <div
                  v-if="!line.noSpeech && line.speaker === '用户' && line.intentZh"
                  class="intent-row"
                >
                  <span class="intent-tag">
                    {{ line.intentZh }}
                  </span>
                </div>
              </div>
              <div v-if="currentRow?.hangupTime && currentDialogLines.length" class="dialog-end-line">
                {{ formatHangupTime(currentRow.hangupTime) }} {{ getHangupDisplayName(currentRow.hangupBy) }}挂断
              </div>
            </div>
          </div>

        </div>

        <div class="detail-right">
          <div class="section">
            <div class="section-title">通话信息</div>
            <div class="info-grid-vertical">
              <div
                v-for="item in currentInfoItems"
                :key="item.label"
                class="info-row"
              >
                <span class="info-label">{{ item.label }}：</span>
                <span class="info-value">{{ item.value }}</span>
              </div>
            </div>
          </div>

        </div>
      </div>

      <div class="drawer-footer">
        <button
          class="btn btn-primary"
          :disabled="isTranslating"
          @click="onTranslate"
        >
          {{ isTranslating ? '翻译中...' : (isTranslated ? '还原' : '翻译') }}
        </button>
        <button class="btn btn-secondary" @click="closeDrawer">
          关闭
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const isTranslated = ref(false)
const isTranslating = ref(false)
const uidKeyword = ref('')
const strategyKeyword = ref('')
const phoneKeyword = ref('')
const roundKeyword = ref('')
const projectKeyword = ref('')
const touchModeKeyword = ref('')

const callData = ref([
  {
    uid: '6550e5669d24c56938a0f43a',
    taskName: 'btsk-c-3',
    strategy: 'collection_bantusk_c3_ivr_c',
    strategyType: '催收',
    round: 7,
    touchTarget: '本人',
    touchMode: '机器人',
    robotName: '智能催收机器人-A',
    scriptName: '',
    userName: 'FENNY PERMATA SARY',
    phone: '81123455855',
    audioEn: '/audio/call1-en.m4a',
    audioZh: '/audio/call1-zh.m4a',
    hangupBy: '用户',
    hangupTime: '2026-02-19 10:23:01',
    basicInfo: {
      项目名称: '催收项目一',
      通话时间: '2026-02-19 10:21:33',
      通话时长: '03:22',
      通话结果: '已接通',
      轮次: '第 7 轮',
      号码归属地: '印尼',
      挂断方: '用户',
      挂断时间: '2026-02-19 10:23:01'
    },
    collectInfo: {
      公司名称关键词: '汇成',
      公司名称: '上海汇成咨询有限公司',
      客户诉求:
        '客户希望在系统中导出所有客户信息，用于后续企业服务及营销使用，需要确认导出操作路径。'
    },
    dialog: [
      {
        speaker: 'AI',
        nodeName: '打招呼',
        time: '10:21:33',
        textEn: 'Hello, this is our intelligent voice service. Are you FENNY?',
        textZh: '您好，这里是智能语音服务，请问是 FENNY 本人吗？'
      },
      {
        speaker: '用户',
        time: '10:21:38',
        noSpeech: true,
        silentSeconds: 5
      },
      {
        speaker: 'AI',
        nodeName: '二次唤醒',
        time: '10:21:45',
        textEn: 'Hello, are you still on the line? If this is not a good time, I can call you back later.',
        textZh: '您好，还在听吗？如果现在不方便，我可以稍后再联系您。'
      },
      {
        speaker: '用户',
        time: '10:21:52',
        textEn: 'I am here. Please tell me the overdue amount first.',
        textZh: '我在听，你先告诉我逾期金额是多少。',
        intentEn: 'Ask overdue amount',
        intentZh: '询问逾期金额'
      },
      {
        speaker: 'AI',
        nodeName: '告知欠款',
        time: '10:22:03',
        textEn:
          'Your current overdue amount is 1,250,000 rupiah. To avoid affecting your credit record, please arrange repayment today.',
        textZh:
          '您当前逾期金额为 1,250,000 印尼盾。为避免影响信用记录，请您今天内安排还款。'
      },
      {
        speaker: '用户',
        time: '10:22:11',
        textEn: 'I am in a meeting now. Can I complete the payment this evening?',
        textZh: '我现在在开会，今晚完成还款可以吗？',
        intentEn: 'Request payment extension',
        intentZh: '申请延后还款'
      },
      {
        speaker: 'AI',
        nodeName: '还款协商',
        time: '10:22:18',
        textEn:
          'Yes. Please complete payment before 21:00 tonight. I can send you the repayment link by SMS now.',
        textZh:
          '可以，请在今晚 21:00 前完成还款。我现在可以通过短信把还款链接发给您。'
      },
      {
        speaker: '用户',
        time: '10:22:22',
        textEn:
          'Okay, please send it. I will process the payment after work and finish it before nine.',
        textZh:
          '好的，你发给我吧，我下班后处理，九点前还上。',
        intentEn: 'Promise to repay',
        intentZh: '承诺还款'
      },
      {
        speaker: 'AI',
        nodeName: '结束确认',
        time: '10:22:30',
        textEn:
          'Thank you for your confirmation. Please keep your phone available for the payment reminder message.',
        textZh:
          '感谢您的确认，请保持手机畅通，稍后会收到还款提醒短信。'
      }
    ]
  },
  {
    uid: '69c0605b875844fe194c3b22',
    taskName: 'btsk-c-3',
    strategy: 'collection_bantusk_c3_ivr_c',
    strategyType: '催收',
    round: 7,
    touchTarget: '联系人',
    touchMode: 'IVR',
    robotName: '智能催收机器人-B',
    scriptName: 'C3-逾期提醒标准话术',
    userName: 'AMIRA NUR HABIB NASUTION',
    phone: '81166225888',
    audioEn: '/audio/call2-en.m4a',
    audioZh: '/audio/call2-zh.m4a',
    hangupBy: '系统',
    hangupTime: '2026-02-19 11:03:30',
    basicInfo: {
      项目名称: '催收项目二',
      通话时间: '2026-02-19 11:03:14',
      通话时长: '02:10',
      通话结果: '未接通',
      轮次: '第 7 轮',
      号码归属地: '印尼',
      挂断方: '系统',
      挂断时间: '2026-02-19 11:03:30'
    },
    dialog: [
      {
        speaker: 'AI',
        nodeName: '打招呼',
        time: '11:03:14',
        textEn:
          'Hello, this is our intelligent voice service. You have a bill that is about to be overdue.',
        textZh: '您好，这里是智能语音服务，您有一笔账单即将到期。'
      }
    ]
  },
  {
    uid: '80f3a0aa91f7436fb244d001',
    taskName: 'btsk-c-3',
    strategy: 'collection_bantusk_c3_ivr_c',
    strategyType: '营销',
    round: 8,
    touchTarget: '本人',
    touchMode: 'IVR',
    robotName: '营销外呼机器人-01',
    scriptName: '营销-活动通知话术',
    userName: 'RANI PUTRI',
    phone: '81277885855',
    audioEn: '',
    audioZh: '',
    hangupBy: '系统',
    hangupTime: '2026-02-19 12:08:22',
    basicInfo: {
      项目名称: '催收项目三',
      通话时间: '2026-02-19 12:08:22',
      通话时长: '00:00',
      通话结果: '无对话内容',
      轮次: '第 8 轮',
      号码归属地: '印尼',
      挂断方: '系统',
      挂断时间: '2026-02-19 12:08:22'
    },
    dialog: []
  }
])

const showDrawer = ref(false)
const currentRow = ref(null)
const copiedCallId = ref(false)
let copyResetTimer = null

const currentCallId = computed(() => currentRow.value?.uid || '--')
const filterVersion = ref(0)

const filteredCallData = computed(() => {
  filterVersion.value
  const uid = uidKeyword.value.trim().toLowerCase()
  const strategy = strategyKeyword.value.trim().toLowerCase()
  const phone = phoneKeyword.value.trim().toLowerCase()
  const round = roundKeyword.value.trim()
  const project = projectKeyword.value.trim()
  const touchMode = touchModeKeyword.value.trim()

  return callData.value.filter((item) => {
    const uidOk = !uid || item.uid.toLowerCase().includes(uid)
    const strategyOk = !strategy || item.strategy.toLowerCase().includes(strategy)
    const phoneOk = !phone || item.phone.toLowerCase().includes(phone)
    const roundOk = !round || String(item.round).includes(round)
    const projectOk = !project || String(item.basicInfo?.项目名称 || '').includes(project)
    const touchModeOk = !touchMode || item.touchMode === touchMode
    return uidOk && strategyOk && phoneOk && roundOk && projectOk && touchModeOk
  })
})

function onSearch() {
  filterVersion.value += 1
}

function onReset() {
  uidKeyword.value = ''
  strategyKeyword.value = ''
  phoneKeyword.value = ''
  roundKeyword.value = ''
  projectKeyword.value = ''
  touchModeKeyword.value = ''
  filterVersion.value += 1
}

function openDrawer(row) {
  currentRow.value = row
  showDrawer.value = true
  isTranslated.value = false
  copiedCallId.value = false
}

function closeDrawer() {
  showDrawer.value = false
  copiedCallId.value = false
}

function getLineText(line) {
  return isTranslated.value ? line.textZh : line.textEn
}

function getAgentLineText(line) {
  const text = getLineText(line)
  const nodeName = line?.nodeName || '节点'
  return `【${nodeName}】${text}`
}

function formatHangupTime(rawTime) {
  if (!rawTime) return ''
  const parts = String(rawTime).split(' ')
  return parts.length > 1 ? parts[1] : parts[0]
}

function formatPhone(rawPhone) {
  if (!rawPhone) return '--'
  const digits = String(rawPhone).replace(/\D/g, '')
  if (digits.length >= 7) {
    return `${digits.slice(0, 3)}****${digits.slice(-4)}`
  }
  return String(rawPhone)
}

const currentAudioUrl = computed(() => {
  if (!currentRow.value) return ''
  return currentRow.value.audioEn || currentRow.value.audioZh || ''
})

const currentDialogLines = computed(() => {
  const dialog = currentRow.value?.dialog || []
  const visibleLines = dialog.filter((line) => !line.noSpeech)
  const normalizedLines = []

  for (const line of visibleLines) {
    const prev = normalizedLines[normalizedLines.length - 1]
    if (!prev || prev.speaker !== line.speaker) {
      normalizedLines.push(line)
    }
  }

  return normalizedLines
})

function getIntentTag(row) {
  if (!row?.dialog?.length) return '--'
  const userIntents = row.dialog
    .filter((line) => line.speaker === '用户' && line.intentZh)
    .map((line) => line.intentZh.trim())
    .filter(Boolean)
  if (!userIntents.length) return '--'
  return userIntents[userIntents.length - 1]
}

function getDialogRound(row) {
  const userReplyCount = (row?.dialog || [])
    .filter((line) => line.speaker === '用户' && !line.noSpeech && (line.textEn || line.textZh))
    .length
  if (!userReplyCount) return '--'
  return `${userReplyCount}轮`
}

function getHangupDisplayName(hangupBy) {
  if (hangupBy === '用户') return '用户'
  return 'AI'
}

function isCallConnected(row) {
  return row?.basicInfo?.通话结果 === '已接通'
}

const currentInfoItems = computed(() => {
  const row = currentRow.value
  if (!row) return []

  const basicInfo = row.basicInfo || {}
  const startCallTime = basicInfo.通话时间 || '--'
  const answerTime = isCallConnected(row) ? startCallTime : '--'

  return [
    { label: '项目名称', value: basicInfo.项目名称 || '--' },
    { label: '任务名称', value: row.taskName || '--' },
    { label: '策略名称', value: row.strategy || '--' },
    { label: '策略类型', value: row.strategyType || '--' },
    { label: '触达轮次', value: row.round ? `第 ${row.round} 轮` : '--' },
    { label: '触达对象', value: row.touchTarget || '--' },
    { label: '触达方式', value: row.touchMode || '--' },
    { label: '机器人名称', value: row.robotName || '--' },
    {
      label: '话术名称',
      value: row.touchMode === 'IVR' ? '催收话术' : (row.touchMode === '机器人' ? (row.robotName || '--') : '--')
    },
    { label: '用户姓名', value: row.userName || '--' },
    { label: '电话号码', value: formatPhone(row.phone) },
    { label: '开始呼叫时间', value: startCallTime },
    { label: '接通时间', value: answerTime },
    { label: '通话时长', value: basicInfo.通话时长 || '--' },
    { label: '对话轮次', value: getDialogRound(row) },
    { label: '意向标签', value: getIntentTag(row) }
  ]
})

function markCopied() {
  copiedCallId.value = true
  if (copyResetTimer) {
    clearTimeout(copyResetTimer)
  }
  copyResetTimer = setTimeout(() => {
    copiedCallId.value = false
  }, 1500)
}

function fallbackCopyText(text) {
  const textarea = document.createElement('textarea')
  textarea.value = text
  textarea.style.position = 'fixed'
  textarea.style.opacity = '0'
  document.body.appendChild(textarea)
  textarea.focus()
  textarea.select()
  const ok = document.execCommand('copy')
  document.body.removeChild(textarea)
  return ok
}

async function copyCurrentCallId() {
  if (!currentCallId.value || currentCallId.value === '--') return
  try {
    if (navigator?.clipboard?.writeText) {
      await navigator.clipboard.writeText(currentCallId.value)
      markCopied()
      return
    }
  } catch (error) {}
  if (fallbackCopyText(currentCallId.value)) {
    markCopied()
  }
}

function onTranslate() {
  if (isTranslated.value) {
    isTranslated.value = false
    return
  }

  if (isTranslating.value) return
  isTranslating.value = true
  setTimeout(() => {
    isTranslated.value = true
    isTranslating.value = false
  }, 800)
}
</script>

<style scoped>
.layout {
  display: flex;
  min-height: 100vh;
  width: 100%;
  background: #f5f6f8;
  --theme-color: #138a73;
  --theme-color-dark: #0f6f5d;
  --theme-color-soft: #e9f6f2;
  --theme-color-soft-text: #2f6f62;
}

.sidebar {
  width: 220px;
  background: #f7f8fa;
  border-right: 1px solid #e9edf2;
  padding: 12px 0;
  flex-shrink: 0;
}

.brand {
  height: 46px;
  padding: 0 18px;
  display: flex;
  align-items: center;
  font-size: 16px;
  font-weight: 600;
  color: #1f2937;
}

.menu-group {
  margin-top: 10px;
}

.menu-title {
  font-size: 13px;
  color: #6b7280;
  padding: 8px 18px;
}

.menu-item {
  height: 34px;
  padding: 0 18px;
  display: flex;
  align-items: center;
  color: #4b5563;
  font-size: 13px;
}

.menu-item.active {
  color: #138a73;
  background: #ebf7f3;
  border-right: 2px solid #138a73;
}

.main-area {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.topbar {
  height: 52px;
  background: #fff;
  border-bottom: 1px solid #e9edf2;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
}

.breadcrumb {
  font-size: 13px;
  color: #6b7280;
}

.user {
  font-size: 13px;
  color: #374151;
}

.content {
  flex: 1;
  padding: 14px 20px;
}

.page-title {
  font-size: 30px;
  font-weight: 600;
  color: #111827;
  margin-bottom: 12px;
}

.card {
  background: #fff;
  border-radius: 6px;
  border: 1px solid #ebeff5;
  box-shadow: none;
  padding: 12px 14px 10px;
}

.filters {
  border: 1px solid #edf1f6;
  border-radius: 4px;
  padding: 10px;
  margin-bottom: 12px;
}

.filter-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  flex-wrap: wrap;
}

.filter-row:last-child {
  margin-bottom: 0;
}

.filter-label {
  font-size: 12px;
  color: #4b5563;
}

.filter-input {
  height: 28px;
  border: 1px solid #d9e0e8;
  border-radius: 3px;
  padding: 0 8px;
  font-size: 12px;
  color: #1f2937;
  background: #fff;
  min-width: 112px;
}

.date-input {
  width: 230px;
}

.uid-input {
  width: 188px;
}

.round-input {
  width: 64px;
}

.touch-mode-select {
  width: 96px;
}

.small-btn {
  height: 28px;
  line-height: 26px;
  padding: 0 12px;
  border-radius: 3px;
}

.table {
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
  font-size: 12px;
}

.table thead {
  background: #f5f7fa;
}

.table th,
.table td {
  padding: 10px 8px;
  border-bottom: 1px solid #eef2f7;
  text-align: left;
  white-space: nowrap;
}

.table tbody tr:hover {
  background: #f5f7fa;
}

.table th {
  color: #374151;
  font-weight: 500;
}

.ellipsis-cell {
  overflow: hidden;
  text-overflow: ellipsis;
}

.action-link {
  border: none;
  background: transparent;
  padding: 0;
  font-size: 12px;
  cursor: pointer;
}

.action-link-detail {
  color: #0f8a73;
}

.action-link-detail:hover {
  color: #0c735f;
}

.action-link-play {
  color: #97a3b6;
}

.action-link-play:hover {
  color: #7b889c;
}

.action-divider {
  color: #c0c8d2;
  font-size: 12px;
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 2px 10px;
  font-size: 12px;
  border-radius: 3px;
  border: 1px solid var(--theme-color);
  color: var(--theme-color);
  background: #fff;
  cursor: pointer;
  transition: all 0.2s;
}

.btn:hover {
  background: var(--theme-color);
  color: #fff;
}

.drawer-mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.35);
  opacity: 0;
  visibility: hidden;
  transition: all 0.25s ease;
  z-index: 1000;
}

.drawer-mask.show {
  opacity: 1;
  visibility: visible;
}

.drawer {
  position: fixed;
  top: 0;
  right: -820px;
  width: 820px;
  height: 100vh;
  background: #f7f9fc;
  box-shadow: -12px 0 36px rgba(15, 23, 42, 0.14);
  transition: right 0.25s ease;
  z-index: 1001;
  display: flex;
  flex-direction: column;
  border-radius: 12px 0 0 12px;
  overflow: hidden;
}

.drawer.show {
  right: 0;
}

.drawer-header {
  padding: 16px 20px;
  border-bottom: 1px solid #e9edf3;
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #ffffff;
}

.drawer-header-left {
  display: flex;
  align-items: center;
  gap: 14px;
  min-width: 0;
}

.drawer-title {
  font-size: 18px;
  font-weight: 600;
  color: #111;
  flex-shrink: 0;
}

.call-id-row {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
}

.call-id-label {
  font-size: 13px;
  color: #6b7280;
  flex-shrink: 0;
}

.call-id-value {
  max-width: 360px;
  font-size: 14px;
  color: #9ca3af;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.copy-btn {
  border: 1px solid #dbe2ea;
  background: #fff;
  color: #4b5563;
  font-size: 12px;
  border-radius: 6px;
  padding: 3px 8px;
  cursor: pointer;
}

.copy-btn:hover {
  border-color: #91d7c6;
  color: var(--theme-color-dark);
}

.drawer-close {
  font-size: 18px;
  cursor: pointer;
  color: #999;
}

.drawer-close:hover {
  color: #333;
}

.drawer-body {
  padding: 14px 16px;
  overflow-y: auto;
  flex: 1;
  display: flex;
  gap: 12px;
}

.detail-left {
  flex: 2;
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-height: 0;
}

.detail-right {
  flex: 1;
  border-left: none;
  padding-left: 0;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.section {
  margin-bottom: 0;
  background: #fff;
  border: 1px solid #edf1f7;
  border-radius: 12px;
  padding: 12px;
}

.audio-section {
  padding: 8px 12px;
}

.dialog-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.section-title {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 10px;
  color: #1f2937;
}

.audio-section .section-title {
  margin-bottom: 6px;
}

.info-grid-vertical {
  display: flex;
  flex-direction: column;
  gap: 10px;
  font-size: 13px;
}

.info-row {
  display: flex;
  align-items: flex-start;
  padding-bottom: 8px;
  border-bottom: 1px solid #f3f5f8;
}

.info-row:last-child {
  border-bottom: none;
}

.info-label {
  font-size: 12px;
  color: #8a93a3;
  width: 76px;
  flex-shrink: 0;
}

.info-value {
  color: #1f2937;
  margin-left: 4px;
  text-align: left;
  flex: 1;
  word-break: break-all;
}

.audio-wrapper {
  display: flex;
  align-items: center;
  gap: 0;
  padding: 0;
  min-width: 0;
}

.audio-wrapper audio {
  width: 100%;
  height: 44px;
}

.audio-wrapper audio::-webkit-media-controls-enclosure {
  min-height: 44px;
}

.audio-wrapper audio::-webkit-media-controls-panel {
  min-height: 44px;
}

.chat-box {
  flex: 1;
  min-height: 0;
  padding: 4px 2px 0;
  overflow-y: auto;
  overflow-x: hidden;
  display: flex;
  flex-direction: column;
  background: transparent;
  border-radius: 0;
  border: none;
}

.dialog-item {
  margin-bottom: 10px;
}

.dialog-item:last-child {
  margin-bottom: 0;
}

.dialog-time-row {
  font-size: 11px;
  color: #a0a7b4;
  margin-bottom: 4px;
  padding: 0 4px;
}

.dialog-time-agent {
  text-align: left;
  margin-left: 48px;
}

.dialog-time-user {
  text-align: right;
  margin-right: 48px;
}

.dialog-main {
  display: flex;
  align-items: center;
}

.dialog-main-agent {
  justify-content: flex-start;
  padding-right: 44px;
}

.dialog-main-user {
  justify-content: flex-end;
  padding-left: 44px;
}

.dialog-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: var(--theme-color-soft);
  color: var(--theme-color-dark);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 600;
  line-height: 1;
  flex-shrink: 0;
  margin-left: 8px;
}

.dialog-main-agent .dialog-avatar {
  margin-left: 0;
  margin-right: 8px;
}

.dialog-bubble {
  font-size: 13px;
  line-height: 1.6;
  color: #333;
  padding: 10px 12px;
  border-radius: 12px;
  background: #f2f4f8;
  box-shadow: none;
  max-width: 100%;
  text-align: left;
  box-sizing: border-box;
  word-break: break-word;
}

.dialog-bubble-user {
  background: var(--theme-color);
  color: #fff;
}

.dialog-bubble-silent {
  background: var(--theme-color-soft);
  color: var(--theme-color-soft-text);
}

.intent-tag {
  display: inline-block;
  margin-left: 4px;
  padding: 1px 8px;
  border-radius: 999px;
  background: var(--theme-color-soft);
  color: var(--theme-color-soft-text);
  font-size: 11px;
  line-height: 1.6;
}

.intent-row {
  margin-top: 4px;
  text-align: right;
}

.dialog-end-line {
  margin-top: auto;
  padding-top: 0;
  font-size: 12px;
  color: #7c8595;
  text-align: center;
}

.drawer-footer {
  padding: 8px 12px;
  border-top: 1px solid #e9edf3;
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 10px;
  background: #ffffff;
}

.btn-secondary {
  border-color: #d7dde8;
  color: #556176;
}

.btn-secondary:hover {
  background: #f4f7fb;
  color: #2f3b4f;
}

.btn-primary {
  background: var(--theme-color);
  color: #fff;
  border-color: var(--theme-color);
}

.btn-primary:hover {
  background: var(--theme-color-dark);
  border-color: var(--theme-color-dark);
}

.drawer-footer .btn {
  padding: 7px 20px;
  font-size: 13px;
  border-radius: 8px;
  font-weight: 500;
}

.drawer-body::-webkit-scrollbar,
.chat-box::-webkit-scrollbar {
  width: 6px;
}

.drawer-body::-webkit-scrollbar-thumb,
.chat-box::-webkit-scrollbar-thumb {
  background: #d8deea;
  border-radius: 999px;
}

.drawer-body::-webkit-scrollbar-track,
.chat-box::-webkit-scrollbar-track {
  background: transparent;
}
</style>