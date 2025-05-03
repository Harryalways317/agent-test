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
        'Authorization': 'Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRoQ2xhc3MiOiJDb21wYW55IiwiYXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsInNvdXJjZSI6IklOVEVHUkFUSU9OIiwic291cmNlSWQiOiI2N2VlNjc1MmY3NTM2NDdiMWM5YWUwNmUtbWEyazZxNTYiLCJjaGFubmVsIjoiT0FVVEgiLCJwcmltYXJ5QXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsIm9hdXRoTWV0YSI6eyJzY29wZXMiOlsiY2FsZW5kYXJzL2V2ZW50cy53cml0ZSIsImNhbGVuZGFycy53cml0ZSIsImNhbGVuZGFycy9yZXNvdXJjZXMud3JpdGUiLCJjYWxlbmRhcnMvcmVzb3VyY2VzLnJlYWRvbmx5IiwiY29waWxvdC5yZWFkb25seSIsImNvcGlsb3Qud3JpdGUiLCJvYXV0aC53cml0ZSIsIm9hdXRoLnJlYWRvbmx5Il0sImNsaWVudCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsInZlcnNpb25JZCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsImNsaWVudEtleSI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZS1tYTJrNnE1NiIsImFnZW5jeVBsYW4iOiJhZ2VuY3lfbW9udGhseV80OTcifSwiaWF0IjoxNzQ2MjYxMzAzLjgxNSwiZXhwIjoxNzQ2MzQ3NzAzLjgxNX0.bI9xc0mMIu7omuHlhb7XXpeOzjvIY9hV_7tkseNNoFOA_NICi6hQGrRbknNj1JVD0JFCpNByPe_iOwNkdy_KCMzUS6z3qGiw-z88DXdyopOjmJ8_4FQNel9AM6oCX55D4DYUqcYhlPCr43beWyu43BB1R4ZdeWgyPNezcwrtykDeQtGz2A7genc1wDeJhx319G4pCzrkZRFEp6HXp9QRevfOtipRNeGWb0UeC6GoZGxBiSKfpTuIS8TRMbpGhweC2iy_faTJmHKQyeOjKfTi6XIfxzfrwN1dYvDetzxEkxid1ZtYnfKtQY87NZfxfRSLuQtfe_bbnMrT2ZC0UxbcNUz6CdSoi71kB2qpMbiAjXoSRfiGvbpZtBCaweV5yb0cDmAH3tw3qeTP4bYixwuuIirEipAl7M5aOsi6YNhmLUH9AdgJygRv8VqYmokc6pzIDV-FqzSbgwaI9MSMmH6Wzyix4VDkU_pUdU2qOcTXqAfFfmp2DLm15TI_EG6zEHXRdHR_VfqkRRILqrKEnMwD9g4v1iS9jVjlVfvdczjLQt7jatNbIQoYqeVlQhk4T82cwrhjusB-AWXH2wxHeYDeZjjK8vp7jvBJZhg1JgDpDsg4xKp7DtR-ucoRO_ORSXzqCb8fq9Xh5A9AjDNQ3s2yrfa8HpnusxMjV8OUSNxHBi0',
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