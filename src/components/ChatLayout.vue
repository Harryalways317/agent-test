<!-- src/components/ChatLayout.vue -->
<template>
  <div class="flex h-screen bg-background">
    <!-- Sidebar -->
    <div class="w-80 border-r border-border bg-card flex flex-col">
      <!-- New Chat Button -->
      <div class="p-4 border-b border-border">
        <button
          class="w-full bg-[#0f172a] text-white hover:bg-[#1e293b] px-4 py-3 rounded-md flex items-center justify-center gap-2 font-medium"
          @click="createNewChat"
          :disabled="isLoading"
        >
          <i-lucide-plus class="w-4 h-4" />
          New Chat
        </button>
      </div>

      <!-- Chat History -->
      <div class="flex-1 overflow-y-auto py-2">
        <div v-if="isLoading" class="p-4 text-center text-muted-foreground">
          Loading chats...
        </div>
        <div v-else-if="chatHistory.length === 0" class="p-4 text-center text-muted-foreground">
          No chats yet
        </div>
        <div
          v-else
          v-for="chat in chatHistory"
          :key="chat.id"
          class="px-3 py-2 mx-2 rounded-lg hover:bg-[#f1f5f9] cursor-pointer mb-1 transition-colors"
          :class="{ 'bg-[#f1f5f9]': chat.id === selectedChatId }"
          @click="selectChat(chat)"
        >
          <div class="flex items-center gap-3">
            <i-lucide-message-circle class="w-4 h-4 text-muted-foreground flex-shrink-0" />
            <span class="text-sm text-[#334155] truncate">{{ chat.session || 'New Chat' }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Main Content -->
    <div class="flex-1 flex flex-col h-full overflow-hidden bg-white">
      <slot></slot>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface Chat {
  id: string
  session: string
  sessionId: string
  conversationHistory: Array<{
    user: string
    agent: Array<{ text: string }>
  }>
}

const chatHistory = ref<Chat[]>([])
const selectedChatId = ref<string | null>(null)
const isLoading = ref(false)

const fetchChats = async () => {
  isLoading.value = true
  try {
    const response = await fetch('https://staging.services.leadconnectorhq.com/copilot/sessions?companyId=5DP4iH6HLkQsiKESj6rh&page=1&limit=10&userId=rTTL2p40dZSG1EMs2TFQ', {
      headers: {
        'Authorization': 'Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRoQ2xhc3MiOiJDb21wYW55IiwiYXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsInNvdXJjZSI6IklOVEVHUkFUSU9OIiwic291cmNlSWQiOiI2N2VlNjc1MmY3NTM2NDdiMWM5YWUwNmUtbWEyazZxNTYiLCJjaGFubmVsIjoiT0FVVEgiLCJwcmltYXJ5QXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsIm9hdXRoTWV0YSI6eyJzY29wZXMiOlsiY2FsZW5kYXJzL2V2ZW50cy53cml0ZSIsImNhbGVuZGFycy53cml0ZSIsImNhbGVuZGFycy9yZXNvdXJjZXMud3JpdGUiLCJjYWxlbmRhcnMvcmVzb3VyY2VzLnJlYWRvbmx5IiwiY29waWxvdC5yZWFkb25seSIsImNvcGlsb3Qud3JpdGUiLCJvYXV0aC53cml0ZSIsIm9hdXRoLnJlYWRvbmx5Il0sImNsaWVudCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsInZlcnNpb25JZCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsImNsaWVudEtleSI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZS1tYTJrNnE1NiIsImFnZW5jeVBsYW4iOiJhZ2VuY3lfbW9udGhseV80OTcifSwiaWF0IjoxNzQ2MTEzOTYyLjYxNCwiZXhwIjoxNzQ2MjAwMzYyLjYxNH0.Noi--PtuDt_ufddlszfA2Uk9UAQvzBrBBREfK3eaHfRZgCUHdjQeb-vZjpM3W5O0vtKinnLbDBfW-DAg3bFOkT_xRBRdq5z3rTtus-kZ5LDViRFCHZLGLjfWdG4pAk_kTWhnpMrdkV2jCVYEa7VzmDjKMFmB_q9drwSA4ycKjzvLo0QAKed-X-Gs6QUo_88xIv21UTL9BkWmKeVmL79v0l2iKvlj7_ETeKdSlcmYaS6gHZWDiPJJOsSrniotCvSDKyICcHpRrLupD4lrnwbJri28aqIyujZM_Aq8h82CZOapNwS34Yzor3l4yJPDj-jDFApeziTk30Jv7PP-8NMuFQ_ePkBJnKzU1DLQhwBH1Nk8gz4gFjjX-FIc7aFqCcDS_CR-INDL23EAbjUuGO1D6MB0Ua7nJ0MnMF8g8ah916A2NB1_cUd03NEnwMm4hepbaX_701Cfkv2QaqFlEGHDgl05ksfrQmQuxNW8KnB8yB5RIi3_gtSa78Mosyv3iMRpjJyPbCus5EwEV6kyI_QjIPGan1HQ9YdIPBGQNCP0IQAElNFRdOoM35Z2oS8I97FJjvL8lyiEC4SS61y9JfhbHBtvYWnCau2JE9Whh0-rC1eD1LTVbAuP8rXlBAwbky5Jlx7z-pGROL7YLcjNuHhKRvldg1IpzO2k1jIUdlBXCtE',
        'Version': '2021-07-28',
        'accept': 'application/json, text/plain, */*',
      }
    })
    const data = await response.json()
    chatHistory.value = data
  } catch (error) {
    console.error('Error fetching chats:', error)
  } finally {
    isLoading.value = false
  }
}

// Add new chat to history
const addNewChat = (chat: Chat) => {
  chatHistory.value = [chat, ...chatHistory.value]
  selectedChatId.value = chat.id
}

defineExpose({
  addNewChat
})

const createNewChat = () => {
  selectedChatId.value = null
  emit('newChat')
}

const selectChat = (chat: Chat) => {
  selectedChatId.value = chat.id
  emit('selectChat', chat)
}

const emit = defineEmits<{
  (e: 'selectChat', chat: Chat): void
  (e: 'newChat'): void
}>()

onMounted(() => {
  fetchChats()
})
</script> 