<template>
  <div class="chat-layout">
    <div class="history-panel">
      <button class="add-chat-btn" @click="addChat">+ 新增对话</button>
      <!-- 这里放历史记录内容 -->
      <div class="history-title">历史记录</div>
      <!-- 可用 v-for 渲染历史会话列表 -->
      <ul class="history-list">
        <li
          v-for="item in historyList"
          :key="item.id"
          class="history-item"
          :class="{ active: String(item.id) === String(currentChatId) }"
          @click="selectChat(item)"
          style="display: flex; align-items: center; justify-content: space-between;"
        >
          <span style="flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;">{{ item.name }}</span>
          <span @click.stop="handleDelete(item.id)" style="margin-left:8px;cursor:pointer;display:flex;align-items:center;">
            <svg width="14" height="14" viewBox="0 0 1024 1024" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M320 896c0 35.2 28.8 64 64 64h256c35.2 0 64-28.8 64-64V320H320v576zm576-704h-192l-32-64H352l-32 64H128v64h768v-64z" fill="#f56c6c"/>
            </svg>
          </span>
        </li>
      </ul>
    </div>
    <div class="ai-chat-simple" :class="{ 'dark': isDark }">
      <!-- 原有聊天内容不变 -->
      <div class="chat-main">
        <div class="messages" ref="messagesRef">
          <div
            v-for="(message, index) in messages"
            :key="index"
            :class="['message', message.role]"
          >
            <span class="role">
              <template v-if="message.role === 'user'"></template>
              <template v-else>
                <img src="@/assets/logo/ai_log.png" alt="AI Logo" class="ai-logo" />
              </template>
            </span>
            <div class="content" v-if="message.role === 'user'">{{ message.content }}</div>
            <div class="content" v-else v-html="renderMarkdown(message.content)"></div>
          </div>
        </div>
        <div class="input-area">
          <textarea
            v-model="userInput"
            @keydown="handleKeydown"
            placeholder="请输入您的问题..."
            rows="1"
            ref="inputRef"
          ></textarea>
          <button class="send-button" @click="sendMessage" :disabled="isStreaming || !userInput.trim()">
            发送
          </button>
          <button class="stop-button" @click="stopMessage" :disabled="!isStreaming">
            停止
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue'
import { useDark } from '@vueuse/core'
import { sendChatMessage, stopChatMessage, getChatHistoryIds, getChatHistoryDetail, deleteChatHistory } from '@/api/nursing/ai'
import { marked } from 'marked'
import hljs from 'highlight.js'
import 'highlight.js/styles/github.css'

// 配置 marked 使用 highlight.js
marked.setOptions({
  highlight: function (code, lang) {
    if (lang && hljs.getLanguage(lang)) {
      return hljs.highlight(code, { language: lang }).value
    }
    return hljs.highlightAuto(code).value
  }
})

const isDark = useDark()
const messagesRef = ref(null)
const inputRef = ref(null)
const userInput = ref('')
const isStreaming = ref(false)
const messages = ref([])
const currentChatId = ref('') // 当前会话ID，统一使用字符串比较

const historyList = ref([]) // 历史会话列表 [{id, name}]
const persistedIds = ref([]) // 后端已保存的会话ID（用于区分本地新建会话与后端会话）

let abortController = null // 当前流式请求的中断控制器
let stoppedByUser = false // 本次对话是否被用户主动打断

// 生成唯一ID（用于新对话）- 时间戳字符串，最终会作为会话ID保存到后端
const generateTempId = () => {
  return String(Date.now())
}

// 数据转换函数 - 适配后端返回的 {role, content} 格式
const transformHistoryData = (historyData) => {
  if (!historyData || !Array.isArray(historyData)) return []
  //只展示有内容的用户提问与AI回复，过滤空消息，避免出现空白气泡
  return historyData
    .filter(item => item && item.content && String(item.content).trim() !== '')
    .map(item => ({
      role: item.role === 'user' ? 'user' : 'assistant',
      content: item.content
    }))
}

// 判断某个会话ID是否已保存到后端
const isPersisted = (id) => {
  return persistedIds.value.includes(String(id))
}

// 用后端的会话ID数组刷新历史列表（保留已有的会话名称）
const applyHistoryList = (ids) => {
  const list = (Array.isArray(ids) ? ids : []).map(id => {
    const idStr = String(id)
    const exist = historyList.value.find(item => String(item.id) === idStr)
    return {
      id: idStr,
      name: exist ? exist.name : '会话 ' + idStr.slice(-6) //取后6位作为默认名称
    }
  })
  historyList.value = list
  persistedIds.value = list.map(item => item.id)
}

// 开启一个新会话（此时会话ID由前端生成，发送第一条消息后后端会记录该ID）
const startNewChat = () => {
  const tempId = generateTempId()
  historyList.value.unshift({ id: tempId, name: '新会话' })
  currentChatId.value = tempId
  messages.value = []
}

// 用后端返回的会话ID同步列表，并保证当前正在使用（后端还没记录）的新会话仍在列表中
const syncHistoryList = (ids) => {
  const currentName = historyList.value.find(item => String(item.id) === String(currentChatId.value))?.name
  applyHistoryList(ids)
  if (currentChatId.value && !isPersisted(currentChatId.value)) {
    historyList.value.unshift({ id: String(currentChatId.value), name: currentName || '新会话' })
  }
}

onMounted(async () => {
  const response = await getChatHistoryIds()
  syncHistoryList(response.data)
  if (historyList.value.length === 0) {
    // 没有历史记录，创建新对话
    startNewChat()
  } else {
    // 有历史记录，默认选中并加载第一个会话
    await selectChat(historyList.value[0])
  }
})

// 新增对话
const addChat = () => {
  startNewChat()
}

// 切换对话
const selectChat = async (item) => {
  if (!item) return
  currentChatId.value = String(item.id)
  messages.value = []
  // 只有后端已保存的会话才去加载聊天记录
  if (isPersisted(item.id)) {
    const detail = await getChatHistoryDetail(item.id)
    messages.value = transformHistoryData(detail)
  }
}

const scrollToBottom = async () => {
  await nextTick()
  if (messagesRef.value) {
    messagesRef.value.scrollTop = messagesRef.value.scrollHeight
  }
}

const sendMessage = async () => {
  if (isStreaming.value) return
  if (!userInput.value.trim()) return
  // 当前没有会话时（例如历史会话已被全部删除）自动新建一个，保证可以直接提问
  if (!currentChatId.value) {
    startNewChat()
  }
  const prompt = userInput.value.trim()
  messages.value.push({ role: 'user', content: prompt })
  userInput.value = ''
  await scrollToBottom()
  // 先插入AI回复的占位气泡，后续按索引更新内容
  const aiIndex = messages.value.push({ role: 'assistant', content: '' }) - 1
  isStreaming.value = true
  stoppedByUser = false
  abortController = new AbortController()
  try {
    const chatId = currentChatId.value
    const response = await sendChatMessage({ prompt, chatId, signal: abortController.signal })
    if (!response.body) throw new Error('无流式响应')

    // 后端在接收消息时已把该会话ID保存到当前用户的会话列表中，这里标记为已保存
    if (!isPersisted(chatId)) {
      persistedIds.value = persistedIds.value.concat(String(chatId))
    }
    // 用第一个提问作为会话名称
    const currentItem = historyList.value.find(item => String(item.id) === String(chatId))
    if (currentItem && currentItem.name === '新会话') {
      currentItem.name = prompt.length > 20 ? prompt.substring(0, 20) + '...' : prompt
    }

    const reader = response.body.getReader()
    const decoder = new TextDecoder('utf-8')
    let accumulated = ''

    while (true) {
      const { value, done } = await reader.read()
      if (done) break
      // stream:true 保证中文等多字节字符在分包边界不会被截断成乱码
      accumulated += decoder.decode(value, { stream: true })
      messages.value[aiIndex].content = accumulated
      await nextTick()
      await scrollToBottom()
    }
    if (!accumulated) {
      //被打断时还没来得及生成内容，给出提示；否则按异常处理
      messages.value[aiIndex].content = stoppedByUser ? '（已打断）' : '抱歉，没有收到回复，请稍后重试。'
    }
  } catch (e) {
    //用户主动打断导致的 AbortError 不算异常，保留已经生成的内容即可
    if (stoppedByUser) {
      console.log('已打断流式输出，保留已生成的内容')
      //一个字都还没生成就被打断时，给出提示
      if (!messages.value[aiIndex].content) {
        messages.value[aiIndex].content = '（已打断）'
      }
    } else {
      console.error('智能客服对话失败:', e)
      messages.value[aiIndex].content = '抱歉，发生了错误，请稍后重试。'
    }
  } finally {
    abortController = null
    stoppedByUser = false
    isStreaming.value = false
    await scrollToBottom()

    // 会话结束后，刷新历史列表（异步执行，不阻塞UI）
    refreshHistoryList()
  }
}

/**
 * 打断当前流式对话：
 * 1. 先中断本地接收：界面立即恢复，右侧已显示的内容不会被清除
 * 2. 再通知后端停止生成（后端会结束模型输出并补存已生成的内容）
 */
const stopMessage = async () => {
  if (!isStreaming.value) return
  stoppedByUser = true
  if (abortController) {
    abortController.abort()
  }
  try {
    await stopChatMessage(currentChatId.value)
  } catch (e) {
    console.error('通知后端打断流式对话失败:', e)
  }
}

// 处理输入框回车和换行
const handleKeydown = (e) => {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault();
    sendMessage();
  }
  // Shift+Enter 默认行为为换行，无需处理
}

// 渲染markdown为html
const renderMarkdown = (text) => {
  return marked.parse(text || '')
}

// 刷新历史列表
const refreshHistoryList = async () => {
  try {
    const response = await getChatHistoryIds()
    // 只刷新左侧会话列表与选中状态，不覆盖右侧已渲染的对话，
    // 避免刚发送/正在显示的消息被历史记录替换掉
    syncHistoryList(response.data)
  } catch (error) {
    console.error('刷新历史列表失败:', error)
  }
}

// 删除时刷新逻辑
const handleDelete = async (chatId) => {
  const idStr = String(chatId)
  // 只有后端已保存的会话才调用删除接口；本地新建（还没发送过消息）的会话直接移除即可
  if (isPersisted(idStr)) {
    await deleteChatHistory(idStr)
  }
  // 删除的就是当前会话时，先清空当前选中状态：
  // 否则同步列表时它会被当成"后端还没记录的新会话"重新加回列表，导致永远删不掉
  const deletingCurrent = String(currentChatId.value) === idStr
  if (deletingCurrent) {
    currentChatId.value = ''
    messages.value = []
  }
  // 重新获取历史会话id
  const response = await getChatHistoryIds()
  applyHistoryList(response.data)
  // 删除的是当前会话：还有会话就切到第一个，全部删完则保持空（此时可直接提问，发送时会自动新建会话）
  if (deletingCurrent) {
    if (historyList.value.length > 0) {
      await selectChat(historyList.value[0])
    }
  }
}
</script>

<style scoped>
.chat-layout {
  display: flex;
  /* width: 100vw; */
  height:calc( 100vh - 85px);
  background: var(--bg-color, #f5f6fa);
}

.history-panel {
  width: 200px;
  background: #f7f7fa;
  border-right: 1px solid #e0e0e0;
  padding: 2rem 1rem;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.add-chat-btn {
  width: 100%;
  padding: 0.75em 0;
  margin-bottom: 1em;
  background: #007cf0;
  color: #fff;
  border: none;
  border-radius: 0.5em;
  font-size: 1em;
  font-weight: bold;
  cursor: pointer;
  transition: background 0.2s;
}
.add-chat-btn:hover {
  background: #005fa3;
}
.history-item.active {
  background: #e6f0fd;
  font-weight: bold;
}

.history-title {
  font-weight: bold;
  font-size: 1.1em;
  margin-bottom: 1.5rem;
}

.history-list {
  list-style: none;
  padding: 0;
  margin: 0;
  flex: 1;
  overflow-y: auto;
}

.history-item {
  padding: 0.75em 0.5em;
  border-radius: 0.5em;
  cursor: pointer;
  transition: background 0.2s;
}
.history-item:hover {
  background: #e6f0fd;
}
.ai-chat-simple {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: stretch;
  justify-content: flex-start;
  background: var(--bg-color, #f5f6fa);
  box-sizing: border-box;
  padding: 30px;
  font-size: 1.05rem;
}
.chat-main {
  width: 100%;
  /* max-width: 1110px; */
  min-width:800px;
  margin: 0;
  background: #fff;
  display: flex;
  flex-direction: column;
  height: 90vh; /* 放大高度 */
  min-height: 605px;
  overflow: hidden;
  border-radius:8px;
}
.messages {
  flex: 1;
  overflow-y: auto;
  padding: 0.5rem 1rem; /* 放大内边距 */
  box-sizing: border-box;
}
.message {
  width: 100%;
  display: flex;
  margin-bottom: 1rem; /* 放大间距 */
}
.message.user {
  justify-content: flex-end;
}
.message .content {
  /* 内容区域样式 */
}
.message.user .content {
  background: #e6f0fd;
  color: #222;
  border-radius: 1.0em 0.2em 1.0em 1.25em; /* 放大圆角 */
  padding: 0.50em 1.2em; /* 放大气泡内边距 */
  max-width: 90%; /* 放大最大宽度 */
  word-break: break-all;
  display: block;
  text-align: left;
  margin-left: 3rem;
  margin-right: 0;
  font-size: 16px;
}
.message.user .role {
  color: #007CF0;
  font-weight: bold;
  margin-left: 0.5em;
  margin-right: 0;
}
.message.assistant {
  display: flex;
  justify-content: flex-start;
  align-items: flex-start;
}
.message.assistant .content {
  color: #222;
  border-radius: 0.2em 1.25em 1.25em 1.25em; /* 放大圆角 */
  padding: 0.50em 1.2em; /* 放大气泡内边距 */
  max-width: 90%; /* 放大最大宽度 */
  word-break: break-all;
  display: block;
  text-align: left;
  margin-right: 3rem;
  margin-left: 8px;
  font-size: 16px;
  flex: 1;
}
.message.assistant .role {
  color: #ff9800;
  font-weight: bold;
  margin-right: 0;
  flex-shrink: 0;
}
.input-area {
  display: flex;
  gap: 0.5rem; /* 放大间距 */
  align-items: flex-end;
  background: #fafbfc;
  padding: 1.5rem 2rem; /* 放大内边距 */
  border-top: 1px solid #eee;
}
textarea {
  flex: 1;
  resize: none;
  border: none;
  background: transparent;
  padding: 1rem;
  font-size: 16px; /* 放大输入字体 */
  line-height: 24px;
  max-height: 180px;
  border-radius: 0.75rem;
  outline: none;
  min-height: 1.5rem; /* 输入框更高 */
}
.send-button {
  min-width: 80px;
  height: 3rem;
  border: none;
  border-radius: 0.75rem;
  background: #007CF0;
  color: #fff;
  font-weight: bold;
  font-size: 1.1rem;
  cursor: pointer;
  transition: background 0.2s;
}
.send-button:disabled {
  background: #ccc;
  cursor: not-allowed;
}
.stop-button {
  min-width: 80px;
  height: 3rem;
  border: none;
  border-radius: 0.75rem;
  background: #f56c6c;
  color: #fff;
  font-weight: bold;
  font-size: 1.1rem;
  cursor: pointer;
  transition: background 0.2s;
}
.stop-button:hover {
  background: #e04b4b;
}
.stop-button:disabled {
  background: #ccc;
  cursor: not-allowed;
}
.dark .chat-main {
  /* background: #222; */
  color: #333;
  font-size:16px;
  line-height:26px;
}
.dark .input-area {
  /* background: #181818;
  border-top: 1px solid #333; */
}
.dark textarea {
  background: #f3f3f3;
  color: #000;
}
.dark textarea::placeholder{
  color:#ccc;
}
.ai-logo {
  width: 24px;
  height: 24px;
  vertical-align: top;
  border-radius: 6px;
  background: #fff;
}
.content p {
font-size:16px;
}
.content table {
  border-collapse: collapse;
  width: 100%;
  margin: 1em 0;
}
.content th,
.content td {
  border: 1px solid #d0d7de;
  padding: 8px 12px;
  text-align: left;
}
.content th {
  background: #f6f8fa;
  font-weight: bold;
}
.preview-area {
  background: #f6f8fa;
  border-top: 1px solid #e0e0e0;
  padding: 1.2em 2em 1.2em 2em;
  margin: 0;
}
.preview-title {
  font-weight: bold;
  margin-bottom: 0.5em;
  color: #007cf0;
}
.preview-content {
  font-size: 1em;
}
</style>
<style>
@import 'highlight.js/styles/github.css';
.preview-content pre {
  background: #f6f8fa;
  padding: 1em;
  border-radius: 6px;
  overflow-x: auto;
}
.preview-content code {
  font-family: 'Fira Mono', 'Consolas', 'Menlo', monospace;
  font-size: 1em;
}
</style>
