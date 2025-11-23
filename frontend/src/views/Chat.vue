<template>
  <div class="chat-container">
    <header class="chat-header">
      <button @click="goBack" class="back-btn">← Назад</button>
      <h2>{{ chatName }}</h2>
      <div></div>
    </header>

    <div v-if="loading" class="loading">Загрузка...</div>
    <div v-else class="chat-content">
      <div ref="messagesContainer" class="messages-container" @scroll="handleScroll">
        <div
          v-for="message in messages"
          :key="message.id"
          :class="['message', { 'own-message': message.sender_id === currentUserId }]"
        >
          <div class="message-header">
            <strong>{{ getSenderName(message) }}</strong>
            <span class="message-time">{{ formatTime(message.created_at) }}</span>
          </div>
          <div class="message-content">{{ message.content }}</div>
        </div>
      </div>

      <div class="input-container">
        <input
          v-model="newMessage"
          type="text"
          placeholder="Введите сообщение..."
          @keyup.enter="sendMessage"
          @input="handleTyping"
        />
        <button @click="sendMessage" :disabled="!newMessage.trim()">
          Отправить
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, nextTick, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { chatsAPI, messagesAPI, authAPI } from '../services/api'
import { ChatWebSocket } from '../services/websocket'

export default {
  name: 'Chat',
  setup() {
    const route = useRoute()
    const router = useRouter()
    const chatId = parseInt(route.params.chatId)
    const messages = ref([])
    const loading = ref(true)
    const newMessage = ref('')
    const chatName = ref('Чат')
    const currentUserId = ref(null)
    const messagesContainer = ref(null)
    const ws = ref(null)
    const shouldAutoScroll = ref(true)

    const loadChat = async () => {
      try {
        const chat = await chatsAPI.getChat(chatId)
        chatName.value = getChatName(chat)
      } catch (error) {
        console.error('Error loading chat:', error)
      }
    }

    const getChatName = (chat) => {
      if (chat.name) return chat.name
      if (chat.members && chat.members.length > 0) {
        const otherMember = chat.members.find(m => m.user_id !== currentUserId.value)
        if (otherMember) {
          return otherMember.user?.full_name || otherMember.user?.email || 'Пользователь'
        }
      }
      return 'Чат'
    }

    const loadMessages = async () => {
      try {
        const response = await chatsAPI.getMessages(chatId)
        messages.value = response.data || []
        // Устанавливаем флаг автоматической прокрутки
        shouldAutoScroll.value = true
      } catch (error) {
        console.error('Error loading messages:', error)
      } finally {
        loading.value = false
        // Прокручиваем вниз после того, как loading станет false и DOM обновится
        await nextTick()
        // Используем несколько попыток для надежной прокрутки
        setTimeout(() => {
          scrollToBottom(true)
        }, 100)
        setTimeout(() => {
          scrollToBottom(true)
        }, 300)
      }
    }

    const getSenderName = (message) => {
      if (message.sender_id === currentUserId.value) {
        return 'Вы'
      }
      return message.sender?.full_name || message.sender?.email || 'Пользователь'
    }

    const formatTime = (dateString) => {
      const date = new Date(dateString)
      return date.toLocaleTimeString('ru-RU', { hour: '2-digit', minute: '2-digit' })
    }

    const scrollToBottom = async (force = false) => {
      if (!force && !shouldAutoScroll.value) return
      
      await nextTick()
      if (messagesContainer.value) {
        const container = messagesContainer.value
        // Используем несколько попыток для надежной прокрутки
        const scroll = () => {
          if (container) {
            const maxScroll = container.scrollHeight - container.clientHeight
            container.scrollTop = maxScroll > 0 ? maxScroll : container.scrollHeight
          }
        }
        // Прокручиваем сразу
        scroll()
        // И еще раз через небольшую задержку для надежности
        requestAnimationFrame(() => {
          scroll()
          // И еще раз после следующего кадра
          requestAnimationFrame(() => {
            scroll()
            // И еще раз для полной уверенности
            setTimeout(() => {
              scroll()
            }, 50)
          })
        })
      }
    }
    
    // Отслеживаем скролл, чтобы определить, нужно ли автоматически прокручивать
    const handleScroll = () => {
      if (!messagesContainer.value) return
      const container = messagesContainer.value
      const threshold = 100 // пикселей от низа
      const isNearBottom = container.scrollHeight - container.scrollTop - container.clientHeight < threshold
      shouldAutoScroll.value = isNearBottom
    }

    const sendMessage = async () => {
      if (!newMessage.value.trim()) return

      const content = newMessage.value.trim()
      newMessage.value = ''

      // Отправляем через WebSocket если подключен, иначе через API
      if (ws.value && ws.value.ws && ws.value.ws.readyState === WebSocket.OPEN) {
        // Оптимистично добавляем сообщение сразу
        const tempId = `temp-${Date.now()}-${Math.random()}`
        const tempMessage = {
          id: tempId,
          chat_id: chatId,
          sender_id: currentUserId.value,
          sender: { id: currentUserId.value, email: '', full_name: 'Вы' },
          content: content,
          created_at: new Date().toISOString(),
          edited_at: null,
          isTemp: true, // Флаг для временного сообщения
          tempId: tempId, // Уникальный идентификатор для замены
        }
        messages.value.push(tempMessage)
        scrollToBottom()
        
        // Отправляем через WebSocket
        try {
          ws.value.sendMessage(content)
        } catch (error) {
          // Если отправка не удалась, удаляем временное сообщение
          const index = messages.value.findIndex(m => m.tempId === tempId)
          if (index !== -1) {
            messages.value.splice(index, 1)
          }
          console.error('Error sending message via WebSocket:', error)
          // Пытаемся отправить через API как fallback
          try {
            const message = await messagesAPI.sendMessage(chatId, content)
            messages.value.push(message)
            scrollToBottom()
          } catch (apiError) {
            console.error('Error sending message via API:', apiError)
            alert('Ошибка отправки сообщения')
          }
        }
      } else {
        // Если WebSocket не подключен, используем API
        try {
          const message = await messagesAPI.sendMessage(chatId, content)
          messages.value.push(message)
          scrollToBottom()
        } catch (error) {
          console.error('Error sending message:', error)
          alert('Ошибка отправки сообщения')
        }
      }
    }

    const handleTyping = () => {
      if (ws.value) {
        ws.value.sendTyping()
      }
    }

    const setupWebSocket = () => {
      const token = localStorage.getItem('access_token')
      if (!token) return

      ws.value = new ChatWebSocket(
        chatId,
        token,
        (data) => {
          console.log('WebSocket callback received data:', data)
          if (data.type === 'new_message') {
            const message = data.message
            console.log('Processing new message:', message)
            
            // Используем nextTick для обеспечения реактивности Vue
            nextTick(() => {
              // Проверяем, нет ли уже такого сообщения по ID (чтобы избежать дубликатов)
              const existingIndex = messages.value.findIndex(m => m.id === message.id)
              if (existingIndex !== -1) {
                // Сообщение уже есть, обновляем его
                console.log('Updating existing message at index:', existingIndex)
                // Создаем новый массив для правильного обновления реактивности Vue
                const newMessages = [...messages.value]
                newMessages[existingIndex] = message
                messages.value = newMessages
              } else {
                // Ищем временное сообщение с таким же содержимым и отправителем
                // Проверяем, что это наше сообщение (от нас же)
                const isOurMessage = message.sender_id === currentUserId.value
                const tempIndex = messages.value.findIndex(
                  m => m.isTemp && 
                       m.content === message.content && 
                       m.sender_id === message.sender_id &&
                       isOurMessage // Только для наших сообщений
                )
                
                if (tempIndex !== -1) {
                  // Заменяем временное сообщение на реальное
                  console.log('Replacing temp message at index:', tempIndex)
                  // Создаем новый массив для правильного обновления реактивности Vue
                  const newMessages = [...messages.value]
                  newMessages[tempIndex] = message
                  messages.value = newMessages
                } else {
                  // Это новое сообщение от другого пользователя или наше, но без временного
                  console.log('Adding new message', isOurMessage ? 'from us (no temp found)' : 'from other user')
                  // Создаем новый массив для принудительного обновления реактивности
                  messages.value = [...messages.value, message]
                }
              }
              // Прокручиваем вниз после обновления сообщений
              scrollToBottom()
            })
          } else {
            console.log('Received WebSocket message with type:', data.type)
          }
        },
        (error) => {
          console.error('WebSocket error:', error)
        }
      )

      ws.value.connect()
    }

    const goBack = () => {
      router.push('/')
    }

    onMounted(async () => {
      try {
        const user = await authAPI.getCurrentUser()
        currentUserId.value = user.id
        await loadChat()
        await loadMessages()
        setupWebSocket()
        // Дополнительная прокрутка после полной загрузки компонента
        await nextTick()
        setTimeout(() => {
          scrollToBottom(true)
        }, 500)
      } catch (error) {
        console.error('Error initializing chat:', error)
        router.push('/')
      }
    })

    onUnmounted(() => {
      if (ws.value) {
        ws.value.disconnect()
      }
    })

    watch(() => route.params.chatId, async (newChatId) => {
      if (ws.value) {
        ws.value.disconnect()
      }
      loading.value = true
      messages.value = []
      shouldAutoScroll.value = true
      await loadChat()
      await loadMessages()
      setupWebSocket()
      // Дополнительная прокрутка после переключения чата
      await nextTick()
      setTimeout(() => {
        scrollToBottom(true)
      }, 500)
    })
    
    // Отслеживаем изменения в messages для автоматической прокрутки
    watch(messages, () => {
      if (shouldAutoScroll.value) {
        scrollToBottom()
      }
    }, { deep: true })

    return {
      messages,
      loading,
      newMessage,
      chatName,
      currentUserId,
      messagesContainer,
      sendMessage,
      handleTyping,
      getSenderName,
      formatTime,
      goBack,
    }
  },
}
</script>

<style scoped>
.chat-container {
  max-width: 1200px;
  margin: 0 auto;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: white;
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border-bottom: 1px solid #eee;
  background: white;
}

.back-btn {
  padding: 0.5rem 1rem;
  background: #f5f5f5;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.back-btn:hover {
  background: #e0e0e0;
}

.chat-header h2 {
  margin: 0;
  flex: 1;
  text-align: center;
}

.chat-content {
  display: flex;
  flex-direction: column;
  flex: 1;
  overflow: hidden;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  background: #f9f9f9;
}

.message {
  margin-bottom: 1rem;
  padding: 0.75rem;
  background: white;
  border-radius: 8px;
  max-width: 70%;
}

.own-message {
  margin-left: auto;
  background: #4CAF50;
  color: white;
}

.message-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.25rem;
  font-size: 0.85rem;
}

.message-time {
  opacity: 0.7;
}

.message-content {
  word-wrap: break-word;
}

.input-container {
  display: flex;
  padding: 1rem;
  border-top: 1px solid #eee;
  background: white;
  gap: 0.5rem;
}

.input-container input {
  flex: 1;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
}

.input-container input:focus {
  outline: none;
  border-color: #4CAF50;
}

.input-container button {
  padding: 0.75rem 1.5rem;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
}

.input-container button:hover:not(:disabled) {
  background: #45a049;
}

.input-container button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.loading {
  text-align: center;
  padding: 2rem;
  color: #666;
}
</style>

