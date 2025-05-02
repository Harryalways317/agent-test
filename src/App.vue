<script setup lang="ts">
import { ref, nextTick, computed, onMounted, onUnmounted, watch } from 'vue'
import ChatLayout from './components/ChatLayout.vue'

const chatLayoutRef = ref<InstanceType<typeof ChatLayout> | null>(null)

const emit = defineEmits<{
  (e: 'newChatCreated', chat: Chat): void
}>()

interface Message {
  id: number
  content: string
  isUser: boolean
  attachments?: Array<{
    imageUrl: string
    type: string
  }>
}

interface Chat {
  id: string
  session: string
  sessionId: string
  conversationHistory: Array<{
    user: string
    agent: Array<{ 
      text: string
      expectedNextInput?: string[]
    }>
  }>
}

const currentChat = ref<Chat | null>(null)
const newMessage = ref('')
const isLoading = ref(false)
const isNewChat = ref(false)
const isInitialLoading = ref(false)
const messagesContainer = ref<HTMLElement | null>(null)
const imageFile = ref<File | null>(null)
const imagePreview = ref<string | null>(null)
const expectedInput = ref<string | null>(null)
const fileInput = ref<HTMLInputElement | null>(null)

// Watch for expected input type changes
watch(() => currentChat.value?.conversationHistory, (newHistory) => {
  if (newHistory && newHistory.length > 0) {
    const lastMessage = newHistory[newHistory.length - 1]
    if (lastMessage.agent?.[0]?.expectedNextInput) {
      expectedInput.value = lastMessage.agent[0].expectedNextInput[0]
    } else {
      expectedInput.value = null
    }
  }
}, { deep: true })

const currentMessages = computed(() => {
  if (!currentChat.value && !isNewChat.value) return []
  
  if (isNewChat.value) {
    return []
  }
  
  return currentChat.value!.conversationHistory.flatMap((entry, index) => {
    const messages: Message[] = []
    
    // Add user message
    if (entry.user) {
      messages.push({
        id: index * 2,
        content: entry.user,
        isUser: true
      })
    }
    
    // Add AI response
    if (entry.agent?.[0]?.text) {
      messages.push({
        id: index * 2 + 1,
        content: entry.agent[0].text,
        isUser: false
      })
    }
    
    return messages
  })
})

const scrollToBottom = async () => {
  await nextTick()
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

// Watch for route changes to load chat by ID
const initializeFromUrl = async () => {
  const sessionId = window.location.pathname.split('/').pop()
  if (sessionId && sessionId !== 'chat') {
    isInitialLoading.value = true
    try {
      const response = await fetch(`https://staging.services.leadconnectorhq.com/copilot/sessions?companyId=5DP4iH6HLkQsiKESj6rh&sessionId=${sessionId}&userId=MASTERMIND`, {
        headers: {
          'Authorization': 'Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRoQ2xhc3MiOiJDb21wYW55IiwiYXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsInNvdXJjZSI6IklOVEVHUkFUSU9OIiwic291cmNlSWQiOiI2N2VlNjc1MmY3NTM2NDdiMWM5YWUwNmUtbWEyazZxNTYiLCJjaGFubmVsIjoiT0FVVEgiLCJwcmltYXJ5QXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsIm9hdXRoTWV0YSI6eyJzY29wZXMiOlsiY2FsZW5kYXJzL2V2ZW50cy53cml0ZSIsImNhbGVuZGFycy53cml0ZSIsImNhbGVuZGFycy9yZXNvdXJjZXMud3JpdGUiLCJjYWxlbmRhcnMvcmVzb3VyY2VzLnJlYWRvbmx5IiwiY29waWxvdC5yZWFkb25seSIsImNvcGlsb3Qud3JpdGUiLCJvYXV0aC53cml0ZSIsIm9hdXRoLnJlYWRvbmx5Il0sImNsaWVudCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsInZlcnNpb25JZCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsImNsaWVudEtleSI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZS1tYTJrNnE1NiIsImFnZW5jeVBsYW4iOiJhZ2VuY3lfbW9udGhseV80OTcifSwiaWF0IjoxNzQ2MTEzOTYyLjYxNCwiZXhwIjoxNzQ2MjAwMzYyLjYxNH0.Noi--PtuDt_ufddlszfA2Uk9UAQvzBrBBREfK3eaHfRZgCUHdjQeb-vZjpM3W5O0vtKinnLbDBfW-DAg3bFOkT_xRBRdq5z3rTtus-kZ5LDViRFCHZLGLjfWdG4pAk_kTWhnpMrdkV2jCVYEa7VzmDjKMFmB_q9drwSA4ycKjzvLo0QAKed-X-Gs6QUo_88xIv21UTL9BkWmKeVmL79v0l2iKvlj7_ETeKdSlcmYaS6gHZWDiPJJOsSrniotCvSDKyICcHpRrLupD4lrnwbJri28aqIyujZM_Aq8h82CZOapNwS34Yzor3l4yJPDj-jDFApeziTk30Jv7PP-8NMuFQ_ePkBJnKzU1DLQhwBH1Nk8gz4gFjjX-FIc7aFqCcDS_CR-INDL23EAbjUuGO1D6MB0Ua7nJ0MnMF8g8ah916A2NB1_cUd03NEnwMm4hepbaX_701Cfkv2QaqFlEGHDgl05ksfrQmQuxNW8KnB8yB5RIi3_gtSa78Mosyv3iMRpjJyPbCus5EwEV6kyI_QjIPGan1HQ9YdIPBGQNCP0IQAElNFRdOoM35Z2oS8I97FJjvL8lyiEC4SS61y9JfhbHBtvYWnCau2JE9Whh0-rC1eD1LTVbAuP8rXlBAwbky5Jlx7z-pGROL7YLcjNuHhKRvldg1IpzO2k1jIUdlBXCtE',
          'Version': '2021-07-28',
          'accept': 'application/json, text/plain, */*',
        }
      })
      const data = await response.json()
      if (data) {
        currentChat.value = data
        scrollToBottom()
      }
    } catch (error) {
      console.error('Error loading chat:', error)
    } finally {
      isInitialLoading.value = false
    }
  }
}

// Update URL when chat changes
const updateUrl = (chat: Chat | null) => {
  if (chat) {
    window.history.pushState({}, '', `/chat/${chat.sessionId}`)
  } else {
    window.history.pushState({}, '', '/chat')
  }
}

// Initialize from URL on mount
onMounted(() => {
  if (!window.location.pathname.includes('/chat')) {
    window.history.pushState({}, '', '/chat')
  }
  initializeFromUrl()
  
  // Listen for back/forward navigation
  window.addEventListener('popstate', initializeFromUrl)
})

onUnmounted(() => {
  window.removeEventListener('popstate', initializeFromUrl)
})

const handleChatSelect = (chat: Chat) => {
  currentChat.value = chat
  isNewChat.value = false
  updateUrl(chat)
  scrollToBottom()
}

const handleNewChat = () => {
  currentChat.value = null
  isNewChat.value = true
  newMessage.value = ''
  updateUrl(null)
}

const handleImageUpload = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    const file = input.files[0]
    imageFile.value = file
    imagePreview.value = URL.createObjectURL(file)
  }
}

const uploadImage = async (file: File): Promise<string> => {
  const formData = new FormData()
  formData.append('file', file)
  formData.append('companyId', '5DP4iH6HLkQsiKESj6rh')
  
  try {
    const response = await fetch('https://staging.services.leadconnectorhq.com/copilot/upload', {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRoQ2xhc3MiOiJDb21wYW55IiwiYXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsInNvdXJjZSI6IklOVEVHUkFUSU9OIiwic291cmNlSWQiOiI2N2VlNjc1MmY3NTM2NDdiMWM5YWUwNmUtbWEyazZxNTYiLCJjaGFubmVsIjoiT0FVVEgiLCJwcmltYXJ5QXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsIm9hdXRoTWV0YSI6eyJzY29wZXMiOlsiY2FsZW5kYXJzL2V2ZW50cy53cml0ZSIsImNhbGVuZGFycy53cml0ZSIsImNhbGVuZGFycy9yZXNvdXJjZXMud3JpdGUiLCJjYWxlbmRhcnMvcmVzb3VyY2VzLnJlYWRvbmx5IiwiY29waWxvdC5yZWFkb25seSIsImNvcGlsb3Qud3JpdGUiLCJvYXV0aC53cml0ZSIsIm9hdXRoLnJlYWRvbmx5Il0sImNsaWVudCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsInZlcnNpb25JZCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsImNsaWVudEtleSI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZS1tYTJrNnE1NiIsImFnZW5jeVBsYW4iOiJhZ2VuY3lfbW9udGhseV80OTcifSwiaWF0IjoxNzQ2MTEzOTYyLjYxNCwiZXhwIjoxNzQ2MjAwMzYyLjYxNH0.Noi--PtuDt_ufddlszfA2Uk9UAQvzBrBBREfK3eaHfRZgCUHdjQeb-vZjpM3W5O0vtKinnLbDBfW-DAg3bFOkT_xRBRdq5z3rTtus-kZ5LDViRFCHZLGLjfWdG4pAk_kTWhnpMrdkV2jCVYEa7VzmDjKMFmB_q9drwSA4ycKjzvLo0QAKed-X-Gs6QUo_88xIv21UTL9BkWmKeVmL79v0l2iKvlj7_ETeKdSlcmYaS6gHZWDiPJJOsSrniotCvSDKyICcHpRrLupD4lrnwbJri28aqIyujZM_Aq8h82CZOapNwS34Yzor3l4yJPDj-jDFApeziTk30Jv7PP-8NMuFQ_ePkBJnKzU1DLQhwBH1Nk8gz4gFjjX-FIc7aFqCcDS_CR-INDL23EAbjUuGO1D6MB0Ua7nJ0MnMF8g8ah916A2NB1_cUd03NEnwMm4hepbaX_701Cfkv2QaqFlEGHDgl05ksfrQmQuxNW8KnB8yB5RIi3_gtSa78Mosyv3iMRpjJyPbCus5EwEV6kyI_QjIPGan1HQ9YdIPBGQNCP0IQAElNFRdOoM35Z2oS8I97FJjvL8lyiEC4SS61y9JfhbHBtvYWnCau2JE9Whh0-rC1eD1LTVbAuP8rXlBAwbky5Jlx7z-pGROL7YLcjNuHhKRvldg1IpzO2k1jIUdlBXCtE',
        'Version': '2021-07-28',
      },
      body: formData
    })
    const data = await response.json()
    return data.url
  } catch (error) {
    console.error('Error uploading image:', error)
    throw error
  }
}

const sendMessage = async () => {
  if ((!newMessage.value.trim() && !imageFile.value) || isLoading.value) return

  const userMessage = newMessage.value
  newMessage.value = ''
  isLoading.value = true
  let attachments = []

  try {
    // Handle image upload if present
    if (imageFile.value) {
      const imageUrl = await uploadImage(imageFile.value)
      attachments.push({
        imageUrl,
        type: 'image'
      })
    }

    // Create or update chat immediately with user message
    if (isNewChat.value) {
      currentChat.value = {
        id: 'temp-' + Date.now(),
        session: 'New Chat',
        sessionId: '',
        conversationHistory: [{
          user: userMessage || 'Here are the images',
          agent: []
        }]
      }
    } else if (currentChat.value) {
      currentChat.value.conversationHistory.push({
        user: userMessage || 'Here are the images',
        agent: []
      })
    }

    // Clear image after adding to chat
    if (imageFile.value) {
      URL.revokeObjectURL(imagePreview.value!)
      imageFile.value = null
      imagePreview.value = null
    }

    await scrollToBottom()

    const response = await fetch('https://staging.services.leadconnectorhq.com/copilot/input?companyId=5DP4iH6HLkQsiKESj6rh&userId=MASTERMIND', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRoQ2xhc3MiOiJDb21wYW55IiwiYXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsInNvdXJjZSI6IklOVEVHUkFUSU9OIiwic291cmNlSWQiOiI2N2VlNjc1MmY3NTM2NDdiMWM5YWUwNmUtbWEyazZxNTYiLCJjaGFubmVsIjoiT0FVVEgiLCJwcmltYXJ5QXV0aENsYXNzSWQiOiI1RFA0aUg2SExrUXNpS0VTajZyaCIsIm9hdXRoTWV0YSI6eyJzY29wZXMiOlsiY2FsZW5kYXJzL2V2ZW50cy53cml0ZSIsImNhbGVuZGFycy53cml0ZSIsImNhbGVuZGFycy9yZXNvdXJjZXMud3JpdGUiLCJjYWxlbmRhcnMvcmVzb3VyY2VzLnJlYWRvbmx5IiwiY29waWxvdC5yZWFkb25seSIsImNvcGlsb3Qud3JpdGUiLCJvYXV0aC53cml0ZSIsIm9hdXRoLnJlYWRvbmx5Il0sImNsaWVudCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsInZlcnNpb25JZCI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZSIsImNsaWVudEtleSI6IjY3ZWU2NzUyZjc1MzY0N2IxYzlhZTA2ZS1tYTJrNnE1NiIsImFnZW5jeVBsYW4iOiJhZ2VuY3lfbW9udGhseV80OTcifSwiaWF0IjoxNzQ2MTEzOTYyLjYxNCwiZXhwIjoxNzQ2MjAwMzYyLjYxNH0.Noi--PtuDt_ufddlszfA2Uk9UAQvzBrBBREfK3eaHfRZgCUHdjQeb-vZjpM3W5O0vtKinnLbDBfW-DAg3bFOkT_xRBRdq5z3rTtus-kZ5LDViRFCHZLGLjfWdG4pAk_kTWhnpMrdkV2jCVYEa7VzmDjKMFmB_q9drwSA4ycKjzvLo0QAKed-X-Gs6QUo_88xIv21UTL9BkWmKeVmL79v0l2iKvlj7_ETeKdSlcmYaS6gHZWDiPJJOsSrniotCvSDKyICcHpRrLupD4lrnwbJri28aqIyujZM_Aq8h82CZOapNwS34Yzor3l4yJPDj-jDFApeziTk30Jv7PP-8NMuFQ_ePkBJnKzU1DLQhwBH1Nk8gz4gFjjX-FIc7aFqCcDS_CR-INDL23EAbjUuGO1D6MB0Ua7nJ0MnMF8g8ah916A2NB1_cUd03NEnwMm4hepbaX_701Cfkv2QaqFlEGHDgl05ksfrQmQuxNW8KnB8yB5RIi3_gtSa78Mosyv3iMRpjJyPbCus5EwEV6kyI_QjIPGan1HQ9YdIPBGQNCP0IQAElNFRdOoM35Z2oS8I97FJjvL8lyiEC4SS61y9JfhbHBtvYWnCau2JE9Whh0-rC1eD1LTVbAuP8rXlBAwbky5Jlx7z-pGROL7YLcjNuHhKRvldg1IpzO2k1jIUdlBXCtE',
        'Version': '2021-07-28',
        'accept': 'application/json, text/plain, */*',
      },
      body: JSON.stringify({
        pageUrl: window.location.href,
        userInput: userMessage || 'Here are the images',
        sessionId: currentChat.value?.sessionId || null,
        attachments,
        inputType: expectedInput.value || "text",
        locationId: "vEoIigWSAw1BQA9DEchD"
      })
    })

    const data = await response.json()
    
    // If this was a new chat, create the chat object
    if (isNewChat.value) {
      const newChat = {
        id: data.id,
        session: data.session,
        sessionId: data.sessionId,
        conversationHistory: data.conversationHistory
      }
      currentChat.value = newChat
      isNewChat.value = false
      
      // Update URL with the new session ID
      updateUrl(newChat)
      
      // Emit event to update sidebar
      chatLayoutRef.value?.addNewChat(newChat)
    } else if (currentChat.value && data.conversationHistory) {
      // Update existing chat with new messages
      currentChat.value.conversationHistory = data.conversationHistory
    }
    
    await scrollToBottom()
  } catch (error) {
    console.error('Error sending message:', error)
    // Add error message to conversation
    if (currentChat.value) {
      currentChat.value.conversationHistory[currentChat.value.conversationHistory.length - 1].agent = [{
        text: "Sorry, there was an error processing your message. Please try again."
      }]
    }
  } finally {
    isLoading.value = false
  }
}

const handleNewChatCreated = (chat: Chat) => {
  chatLayoutRef.value?.addNewChat(chat)
}
</script>

<template>
  <ChatLayout 
    ref="chatLayoutRef"
    @select-chat="handleChatSelect" 
    @new-chat="handleNewChat"
  >
    <div class="flex flex-col h-full">
      <!-- Messages Container -->
      <div class="flex-1 overflow-y-auto bg-white" ref="messagesContainer">
        <div v-if="!currentChat && !isNewChat && !isInitialLoading" class="h-full flex items-center justify-center">
          <div class="text-center text-muted-foreground">
            <p class="text-lg font-medium mb-2">Welcome to Chat</p>
            <p class="text-sm text-gray-500">Select a chat or start a new conversation</p>
          </div>
        </div>
        <div v-else-if="isInitialLoading" class="h-full flex items-center justify-center">
          <div class="text-center text-muted-foreground">
            Loading messages...
          </div>
        </div>
        <template v-else>
          <!-- Fixed width container for messages -->
          <div class="w-[800px] mx-auto py-6">
            <div v-for="message in currentMessages" :key="message.id" class="mb-6 px-4">
              <!-- User Message -->
              <div v-if="message.isUser" class="flex flex-col space-y-2">
                <div class="flex items-start gap-3">
                  <div class="w-8 h-8 rounded-full bg-blue-600 flex items-center justify-center flex-shrink-0">
                    <i-lucide-user class="w-5 h-5 text-white" />
                  </div>
                  <div class="flex-1 space-y-2">
                    <div class="font-medium text-sm text-gray-900">You</div>
                    <div class="text-gray-800">{{ message.content }}</div>
                    <!-- Display attached images -->
                    <div v-if="message.attachments?.length" class="mt-2 space-y-2">
                      <img 
                        v-for="(attachment, index) in message.attachments" 
                        :key="index"
                        :src="attachment.imageUrl" 
                        :alt="'Uploaded image ' + (index + 1)"
                        class="max-h-64 rounded-lg"
                      />
                    </div>
                  </div>
                </div>
              </div>
              
              <!-- AI Message -->
              <div v-else class="flex flex-col space-y-2">
                <div class="flex items-start gap-3">
                  <div class="w-8 h-8 rounded-full bg-[#0f172a] flex items-center justify-center flex-shrink-0">
                    <i-lucide-bot class="w-5 h-5 text-white" />
                  </div>
                  <div class="flex-1 space-y-2">
                    <div class="font-medium text-sm text-gray-900">Assistant</div>
                    <div class="text-gray-800 prose prose-sm max-w-none">
                      {{ message.content }}
                    </div>
                    <!-- Show image upload prompt if needed -->
                    <div v-if="message.agent?.[0]?.expectedNextInput?.includes('image')" class="mt-2 text-sm text-blue-600">
                      Please upload an image to continue
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- Loading Indicator -->
            <div v-if="isLoading" class="px-4 flex items-start gap-3">
              <div class="w-8 h-8 rounded-full bg-[#0f172a] flex items-center justify-center flex-shrink-0">
                <i-lucide-bot class="w-5 h-5 text-white" />
              </div>
              <div class="flex-1">
                <div class="font-medium text-sm text-gray-900 mb-2">Assistant</div>
                <div class="text-gray-800">
                  <span class="inline-flex gap-1">
                    <span class="animate-bounce">.</span>
                    <span class="animate-bounce" style="animation-delay: 0.2s">.</span>
                    <span class="animate-bounce" style="animation-delay: 0.4s">.</span>
                  </span>
                </div>
              </div>
            </div>
          </div>
        </template>
      </div>

      <!-- Message Input -->
      <div class="border-t border-gray-200 bg-white">
        <div class="w-[800px] mx-auto px-4 py-4">
          <!-- Image Upload Preview -->
          <div v-if="imagePreview" class="mb-4">
            <div class="relative inline-block">
              <img :src="imagePreview" alt="Preview" class="max-h-48 rounded-lg" />
              <button
                @click="() => { URL.revokeObjectURL(imagePreview); imagePreview = null; imageFile = null; }"
                class="absolute -top-2 -right-2 bg-red-500 text-white rounded-full p-1 hover:bg-red-600"
              >
                <i-lucide-x class="w-4 h-4" />
              </button>
            </div>
          </div>

          <form @submit.prevent="sendMessage" class="space-y-4">
            <!-- Image Upload Area -->
            <div v-if="expectedInput === 'image'" class="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center">
              <input
                type="file"
                accept="image/*"
                class="hidden"
                ref="fileInput"
                @change="handleImageUpload"
              />
              <div v-if="!imagePreview" class="space-y-2">
                <i-lucide-image class="w-12 h-12 mx-auto text-gray-400" />
                <div class="text-sm text-gray-600">
                  <button type="button" @click="$refs.fileInput.click()" class="text-blue-500 hover:text-blue-600">
                    Click to upload
                  </button>
                  or drag and drop
                </div>
                <p class="text-xs text-gray-500">PNG, JPG, GIF up to 10MB</p>
              </div>
            </div>

            <!-- Message Input -->
            <div class="flex gap-3">
              <input
                v-if="expectedInput !== 'image'"
                v-model="newMessage"
                type="text"
                placeholder="Type your message..."
                class="flex-1 min-h-[44px] rounded-lg border border-gray-200 bg-white px-3 py-2 text-sm ring-offset-background placeholder:text-gray-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-blue-500 focus-visible:ring-offset-2"
                @keydown.enter.prevent="sendMessage"
                :disabled="isLoading"
              />
              <button
                type="submit"
                class="bg-[#0f172a] text-white hover:bg-[#1e293b] px-4 py-2 rounded-lg flex items-center justify-center disabled:opacity-50 disabled:cursor-not-allowed min-h-[44px] min-w-[44px]"
                :disabled="isLoading || (expectedInput === 'image' && !imageFile)"
              >
                <i-lucide-send class="w-5 h-5" />
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  </ChatLayout>
</template>

<style>
@import '@/assets/main.css';

.prose {
  max-width: none;
}
.prose p {
  margin-top: 0.5em;
  margin-bottom: 0.5em;
}
</style>
