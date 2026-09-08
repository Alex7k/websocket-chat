<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref } from 'vue'
import { io, type Socket } from 'socket.io-client'

type ChatMessage = {
  id: string
  text: string
  username: string
  displayName: string
  createdAt: string
}

const DISPLAY_NAME_STORAGE_KEY = 'anon-chat-display-name'
const MAX_TEXT_LENGTH = 1000
const API_BASE = import.meta.env.VITE_API_URL ?? '/api'
const SOCKET_PATH = import.meta.env.VITE_SOCKET_PATH ?? '/api/socket.io'

const message_sound = new Audio('message.mp3')
const messages = ref<ChatMessage[]>([])
const messageText = ref('')
const username = ref('')
const displayName = ref('')
const errorMessage = ref('')
const isSending = ref(false)
const connectionState = ref<'connecting' | 'connected' | 'disconnected'>('connecting')
const onlineCount = ref(0)
const listElement = ref<HTMLElement | null>(null)

let socket: Socket | null = null

const charCount = computed(() => messageText.value.length)
const statusText = computed(() => {
  if (connectionState.value === 'connected') {
    return `${onlineCount.value} online`
  }

  return connectionState.value
})

async function loadIdentity() {
  const savedDisplayName = localStorage.getItem(DISPLAY_NAME_STORAGE_KEY)?.trim()
  const response = await fetch(`${API_BASE}/identity`, {
    credentials: 'include',
  })
  if (!response.ok) {
    throw new Error('Could not initialize identity')
  }

  const payload = (await response.json()) as { username: string }
  const serverUsername = payload.username.trim().slice(0, 64)
  if (!serverUsername) {
    throw new Error('Invalid identity response')
  }

  username.value = serverUsername

  displayName.value = (savedDisplayName || username.value).slice(0, 64)
  localStorage.setItem(DISPLAY_NAME_STORAGE_KEY, displayName.value)
}

function saveDisplayName() {
  const cleanName = displayName.value.trim().slice(0, 64)
  displayName.value = cleanName || username.value
  localStorage.setItem(DISPLAY_NAME_STORAGE_KEY, displayName.value)
}

function getAvatarInitials(rawUsername: string) {
  const compact = rawUsername.replace(/[^a-zA-Z0-9]/g, '').toUpperCase()
  return compact.slice(0, 2) || 'AN'
}

function avatarStyle(rawUsername: string) {
  let hash = 0
  for (let i = 0; i < rawUsername.length; i += 1) {
    hash = (hash * 31 + rawUsername.charCodeAt(i)) % 360
  }
  // Stay inside the red band of the site palette; vary tone, not hue.
  const hue = (hash % 16) - 6
  const lightness = 20 + (hash % 5) * 7
  return { background: `hsl(${hue} 72% ${lightness}%)` }
}

async function scrollToBottom() {
  await nextTick()
  if (!listElement.value) {
    return
  }

  listElement.value.scrollTop = listElement.value.scrollHeight
}

async function loadMessages() {
  const response = await fetch(`${API_BASE}/messages?limit=200`, {
    credentials: 'include',
  })
  if (!response.ok) {
    throw new Error('Could not load messages')
  }

  const payload = (await response.json()) as { messages: ChatMessage[] }
  messages.value = payload.messages
  await scrollToBottom()
}

function sleep(ms: number) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms)
  })
}

async function loadMessagesWithRetry(attempts = 5, delayMs = 600) {
  let lastError: unknown

  for (let i = 0; i < attempts; i += 1) {
    try {
      await loadMessages()
      return
    } catch (error) {
      lastError = error
      if (i < attempts - 1) {
        await sleep(delayMs)
      }
    }
  }

  throw lastError
}

function connectSocket() {
  socket = io(API_BASE === '/api' ? undefined : API_BASE, {
    path: SOCKET_PATH,
    transports: ['websocket', 'polling'],
    withCredentials: true,
  })

  socket.on('connect', async () => {
    connectionState.value = 'connected'

    // If initial history failed during backend startup, recover on socket connect.
    if (messages.value.length === 0) {
      try {
        await loadMessages()
        errorMessage.value = ''
      } catch {
        // Keep existing UX; user can still receive live messages.
      }
    }
  })

  socket.on('disconnect', () => {
    connectionState.value = 'disconnected'
    onlineCount.value = 0
  })

  socket.on('presence:count', (count: unknown) => {
    if (typeof count === 'number' && Number.isFinite(count) && count >= 0) {
      onlineCount.value = Math.floor(count)
    }
  })

  socket.on('messages:new', async (message: ChatMessage) => {
    messages.value.push(message)
    if (message.username !== username.value) {
      void message_sound.play()
    }
    await scrollToBottom()
  })
}

async function sendMessage() {
  errorMessage.value = ''
  const cleanText = messageText.value.trim()

  if (!cleanText) {
    errorMessage.value = 'Message cannot be empty.'
    return
  }

  if (cleanText.length > MAX_TEXT_LENGTH) {
    errorMessage.value = `Message must be ${MAX_TEXT_LENGTH} characters or less.`
    return
  }

  isSending.value = true
  try {
    const response = await fetch(`${API_BASE}/messages`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        text: cleanText,
        displayName: displayName.value,
      }),
    })

    if (!response.ok) {
      const payload = (await response.json()) as { message?: string }
      throw new Error(payload.message ?? 'Message send failed')
    }

    messageText.value = ''
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : 'Message send failed'
  } finally {
    isSending.value = false
  }
}

function onComposerKeyDown(event: KeyboardEvent) {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    void sendMessage()
  }
}

function formatTimestamp(timestamp: string) {
  const messageDate = new Date(timestamp)
  const now = new Date()
  const isToday =
    messageDate.getFullYear() === now.getFullYear() &&
    messageDate.getMonth() === now.getMonth() &&
    messageDate.getDate() === now.getDate()

  const year = messageDate.getFullYear()
  const month = String(messageDate.getMonth() + 1).padStart(2, '0')
  const day = String(messageDate.getDate()).padStart(2, '0')

  const rawHours = messageDate.getHours()
  const minutes = String(messageDate.getMinutes()).padStart(2, '0')
  const period = rawHours >= 12 ? 'PM' : 'AM'
  const hour12 = rawHours % 12 === 0 ? 12 : rawHours % 12
  const hours = String(hour12).padStart(2, '0')

  if (isToday) {
    return `${hours}:${minutes} ${period}`
  }

  return `${year}-${month}-${day} ${hours}:${minutes} ${period}`
}

onMounted(async () => {
  try {
    await loadIdentity()
    await loadMessagesWithRetry()
  } catch {
    errorMessage.value = 'Could not initialize chat.'
  }
  connectSocket()
})

onUnmounted(() => {
  socket?.disconnect()
})
</script>

<template>
  <main class="chat-app">
    <header class="topbar">
      <h1>Alex7k Chatroom</h1>
      <div class="topbar-right">
        <label class="identity-field">
          <span>Display Name</span>
          <input id="displayName" v-model="displayName" maxlength="64" @change="saveDisplayName" />
        </label>
        <p class="username-tag">@{{ username }}</p>
        <p class="status" :class="`status-${connectionState}`">{{ statusText }}</p>
      </div>
    </header>

    <section ref="listElement" class="messages">
      <p v-if="messages.length === 0" class="empty">No messages yet.</p>
      <article v-for="message in messages" :key="message.id" class="message">
        <div class="avatar" :style="avatarStyle(message.username)" aria-hidden="true">
          {{ getAvatarInitials(message.username) }}
        </div>
        <div class="message-body">
          <p class="meta">
            <strong>{{ message.displayName }}</strong>
            <span v-if="message.displayName !== message.username" class="meta-username"
              >@{{ message.username }}</span
            >
            <time :datetime="message.createdAt">{{ formatTimestamp(message.createdAt) }}</time>
          </p>
          <p class="text">{{ message.text }}</p>
        </div>
      </article>
    </section>

    <form class="composer" @submit.prevent="sendMessage">
      <textarea
        v-model="messageText"
        :maxlength="MAX_TEXT_LENGTH"
        rows="3"
        placeholder="Write a message..."
        @keydown="onComposerKeyDown"
      />
      <div class="composer-row">
        <p class="count">{{ charCount }}/{{ MAX_TEXT_LENGTH }}</p>
        <button type="submit" :disabled="isSending">{{ isSending ? 'Sending...' : 'Send' }}</button>
      </div>
      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </form>
  </main>
</template>

<style scoped>
:global(:root) {
  color-scheme: dark;

  --bg: #000000;
  --text: #ff2d2d;
  --muted: #d14545;
  --rule: #5e1414;
  --surface: #150303;
  --surface-hover: #240606;
  --link: #ff6b6b;
  --link-hover: #ffa5a5;
  --error: #ff3b3b;
  --ok: #ff9d9d;
  --message: #ffffff;
}

:global(*) {
  box-sizing: border-box;
  /* Sharp corners everywhere, matching alex7k.com. */
  border-radius: 0;
}

:global(html) {
  background: var(--bg);
}

:global(body) {
  margin: 0;
  min-height: 100vh;
  font-family:
    ui-monospace, 'Cascadia Code', 'SF Mono', Menlo, Consolas, 'Liberation Mono', monospace;
  font-size: 16px;
  line-height: 1.6;
  color: var(--text);
  /* No background here: the backdrop sits at z-index -1, behind body's own
     background but in front of html's, so painting body would hide it. */
}

.chat-app {
  display: grid;
  grid-template-rows: auto 1fr auto;
  height: 100vh;
  width: 100vw;
  padding: 16px;
  gap: 16px;
  overflow: hidden;
}

.topbar {
  border-bottom: 1px solid var(--rule);
  padding-bottom: 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
}

.topbar h1 {
  margin: 0;
  font-size: 20px;
  font-weight: 700;
  letter-spacing: -0.02em;
  color: var(--text);
}

.topbar h1::before {
  content: '~/';
  color: var(--muted);
  font-weight: 400;
}

.topbar-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.identity-field {
  display: flex;
  align-items: center;
  gap: 8px;
}

.identity-field span {
  font-size: 13px;
  text-transform: lowercase;
  color: var(--muted);
}

.identity-field span::after {
  content: ':';
}

.identity-field input {
  width: 14rem;
  border: 1px solid var(--rule);
  background: var(--surface);
  color: var(--text);
  font: inherit;
  font-size: 14px;
  padding: 5px 8px;
}

.identity-field input:focus {
  outline: none;
  border-color: var(--text);
}

.username-tag {
  margin: 0;
  /* Pills, per request: these two keep the rounded shape from the old design. */
  border: 1px solid var(--rule);
  border-radius: 999px;
  background: var(--surface);
  color: var(--muted);
  font-size: 13px;
  padding: 5px 12px;
}

.status {
  margin: 0;
  font-size: 13px;
  border: 1px solid var(--rule);
  border-radius: 999px;
  background: var(--surface);
  padding: 5px 12px;
}

/* Brackets track the state colour so they read as part of the label. */
.status::before {
  content: '[';
  color: currentColor;
  opacity: 0.65;
}

.status::after {
  content: ']';
  color: currentColor;
  opacity: 0.65;
}

.status-connected {
  color: var(--ok);
}

.status-connecting,
.status-disconnected {
  color: var(--error);
}

.messages {
  border: 1px solid var(--rule);
  /* Backdrop is bound to this box, so `contain` rescales it with the panel
     instead of letting it clip past the edges. Layers, front to back:
     red tint, black dimmer (how strongly the art reads), artwork. */
  background:
    linear-gradient(rgba(21, 3, 3, 0.3), rgba(21, 3, 3, 0.3)),
    linear-gradient(rgba(0, 0, 0, 0.78), rgba(0, 0, 0, 0.78)),
    url('/skynet.png') center / contain no-repeat;
  min-height: 0;
  overflow-y: auto;
  padding: 12px;
  scroll-behavior: smooth;
  scrollbar-color: var(--rule) transparent;
}

.empty {
  margin: 8px 0;
  color: var(--muted);
}

.empty::before {
  content: '- ';
  color: var(--rule);
}

.message {
  display: grid;
  grid-template-columns: 2.25rem 1fr;
  gap: 10px;
  padding: 6px 8px;
}

.message:hover {
  background: var(--surface-hover);
}

.avatar {
  width: 2.25rem;
  height: 2.25rem;
  border: 1px solid var(--rule);
  color: #ffdada;
  display: grid;
  place-items: center;
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0.03em;
}

.message-body {
  min-width: 0;
}

.meta {
  display: flex;
  align-items: baseline;
  gap: 8px;
  margin: 0 0 2px 0;
  font-size: 13px;
  color: var(--muted);
}

.meta strong {
  color: var(--text);
  font-size: 14px;
  font-weight: 700;
}

.meta-username {
  color: var(--muted);
}

.text {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-word;
  line-height: 1.5;
  color: var(--message);
}

.composer {
  border-top: 1px solid var(--rule);
  padding-top: 12px;
  display: grid;
  gap: 8px;
}

.composer textarea {
  width: 100%;
  resize: vertical;
  min-height: 3rem;
  max-height: 12rem;
  border: 1px solid var(--rule);
  background: var(--surface);
  color: var(--text);
  font: inherit;
  font-size: 15px;
  padding: 10px 12px;
  line-height: 1.5;
}

.composer textarea::placeholder {
  color: var(--muted);
  opacity: 1;
}

.composer textarea:focus {
  outline: none;
  border-color: var(--text);
}

.composer-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.count {
  margin: 0;
  color: var(--muted);
  font-size: 13px;
}

.composer button {
  border: 1px solid var(--rule);
  background: var(--surface);
  color: var(--text);
  font: inherit;
  font-size: 14px;
  font-weight: 700;
  padding: 7px 18px;
  cursor: pointer;
  transition:
    background 0.15s ease,
    border-color 0.15s ease,
    color 0.15s ease;
}

.composer button:hover:enabled {
  border-color: var(--text);
  background: var(--surface-hover);
  color: var(--link-hover);
}

.composer button:disabled {
  border-color: var(--rule);
  color: var(--muted);
  cursor: not-allowed;
}

.error {
  margin: 0;
  color: var(--error);
  font-size: 13px;
}

@media (max-width: 860px) {
  .chat-app {
    padding: 12px;
    gap: 12px;
  }

  .topbar {
    flex-direction: column;
    align-items: stretch;
    gap: 10px;
  }

  .topbar-right {
    justify-content: space-between;
    flex-wrap: wrap;
  }

  .identity-field {
    flex: 1;
  }

  .identity-field input {
    width: 100%;
  }
}
</style>
