<template>
  <div class="home-container">
    <header class="header">
      <h1>Мессенджер</h1>
      <div class="header-actions">
        <router-link to="/profile" class="profile-link">Профиль</router-link>
        <button @click="handleLogout" class="logout-btn">Выйти</button>
      </div>
    </header>

    <div class="content">
      <div class="search-section">
        <h2>Поиск пользователей</h2>
        <div class="search-box">
          <input
            v-model="searchQuery"
            type="text"
            placeholder="Введите email или имя..."
            @input="handleSearch"
          />
          <div v-if="searching" class="loading">Поиск...</div>
          <div v-if="searchResults.length > 0" class="search-results">
            <div
              v-for="user in searchResults"
              :key="user.id"
              class="user-item"
              @click="startChat(user)"
            >
              <div class="user-info">
                <strong>{{ user.full_name || user.email }}</strong>
                <span class="user-email">{{ user.email }}</span>
              </div>
              <button class="chat-btn">Написать</button>
            </div>
          </div>
          <div v-if="searchQuery && !searching && searchResults.length === 0" class="no-results">
            Пользователи не найдены
          </div>
        </div>
      </div>

      <div class="chats-section">
        <h2>Мои чаты</h2>
        <div v-if="loadingChats" class="loading">Загрузка чатов...</div>
        <div v-else-if="chats.length === 0" class="no-chats">
          У вас пока нет чатов. Найдите пользователя и начните переписку!
        </div>
        <div v-else class="chats-list">
          <div
            v-for="chat in chats"
            :key="chat.id"
            class="chat-item"
            @click="openChat(chat.id)"
          >
            <div class="chat-info">
              <strong>{{ getChatName(chat) }}</strong>
              <span v-if="chat.last_message" class="last-message">
                {{ chat.last_message.content }}
              </span>
            </div>
            <span v-if="chat.last_message" class="chat-time">
              {{ formatTime(chat.last_message.created_at) }}
            </span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { usersAPI, chatsAPI } from '../services/api'

export default {
  name: 'Home',
  setup() {
    const router = useRouter()
    const searchQuery = ref('')
    const searchResults = ref([])
    const searching = ref(false)
    const chats = ref([])
    const loadingChats = ref(false)
    const currentUser = ref(null)

    const handleSearch = async () => {
      if (!searchQuery.value.trim()) {
        searchResults.value = []
        return
      }

      searching.value = true
      try {
        const response = await usersAPI.search(searchQuery.value)
        searchResults.value = response.data || []
      } catch (error) {
        console.error('Search error:', error)
        searchResults.value = []
      } finally {
        searching.value = false
      }
    }

    const startChat = async (user) => {
      try {
        const chat = await chatsAPI.createPrivateChat(user.id)
        router.push(`/chat/${chat.id}`)
      } catch (error) {
        console.error('Error creating chat:', error)
        alert('Ошибка создания чата')
      }
    }

    const openChat = (chatId) => {
      router.push(`/chat/${chatId}`)
    }

    const getChatName = (chat) => {
      if (chat.name) return chat.name
      if (chat.members && chat.members.length > 0) {
        const otherMember = chat.members.find(m => m.user_id !== currentUser.value?.id)
        if (otherMember) {
          return otherMember.user?.full_name || otherMember.user?.email || 'Пользователь'
        }
      }
      return 'Чат'
    }

    const formatTime = (dateString) => {
      const date = new Date(dateString)
      const now = new Date()
      const diff = now - date
      const minutes = Math.floor(diff / 60000)
      
      if (minutes < 1) return 'только что'
      if (minutes < 60) return `${minutes} мин назад`
      const hours = Math.floor(minutes / 60)
      if (hours < 24) return `${hours} ч назад`
      return date.toLocaleDateString()
    }

    const loadChats = async () => {
      loadingChats.value = true
      try {
        const response = await chatsAPI.getChats()
        chats.value = response.data || []
        // Сортируем по времени последнего сообщения
        chats.value.sort((a, b) => {
          const timeA = a.last_message?.created_at || a.updated_at
          const timeB = b.last_message?.created_at || b.updated_at
          return new Date(timeB) - new Date(timeA)
        })
      } catch (error) {
        console.error('Error loading chats:', error)
      } finally {
        loadingChats.value = false
      }
    }

    const handleLogout = () => {
      localStorage.removeItem('access_token')
      router.push('/login')
    }

    onMounted(async () => {
      try {
        const { authAPI } = await import('../services/api')
        currentUser.value = await authAPI.getCurrentUser()
      } catch (error) {
        console.error('Error loading user:', error)
      }
      await loadChats()
    })

    return {
      searchQuery,
      searchResults,
      searching,
      chats,
      loadingChats,
      handleSearch,
      startChat,
      openChat,
      getChatName,
      formatTime,
      handleLogout,
    }
  },
}
</script>

<style scoped>
.home-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 1rem;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background: white;
  border-radius: 8px;
  margin-bottom: 1rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.header h1 {
  margin: 0;
  color: #333;
}

.header-actions {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.profile-link {
  color: #4CAF50;
  text-decoration: none;
  font-weight: 500;
}

.profile-link:hover {
  text-decoration: underline;
}

.logout-btn {
  padding: 0.5rem 1rem;
  background: #f44336;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.logout-btn:hover {
  background: #d32f2f;
}

.content {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.search-section,
.chats-section {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

h2 {
  margin-top: 0;
  margin-bottom: 1rem;
  color: #333;
}

.search-box input {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  box-sizing: border-box;
}

.search-results {
  margin-top: 1rem;
}

.user-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border: 1px solid #eee;
  border-radius: 4px;
  margin-bottom: 0.5rem;
  cursor: pointer;
  transition: background 0.2s;
}

.user-item:hover {
  background: #f5f5f5;
}

.user-info {
  display: flex;
  flex-direction: column;
}

.user-email {
  font-size: 0.9rem;
  color: #666;
}

.chat-btn {
  padding: 0.5rem 1rem;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.chat-btn:hover {
  background: #45a049;
}

.chats-list {
  max-height: 600px;
  overflow-y: auto;
}

.chat-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border: 1px solid #eee;
  border-radius: 4px;
  margin-bottom: 0.5rem;
  cursor: pointer;
  transition: background 0.2s;
}

.chat-item:hover {
  background: #f5f5f5;
}

.chat-info {
  display: flex;
  flex-direction: column;
  flex: 1;
}

.last-message {
  font-size: 0.9rem;
  color: #666;
  margin-top: 0.25rem;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 300px;
}

.chat-time {
  font-size: 0.85rem;
  color: #999;
}

.loading,
.no-results,
.no-chats {
  text-align: center;
  padding: 2rem;
  color: #666;
}

@media (max-width: 768px) {
  .content {
    grid-template-columns: 1fr;
  }
}
</style>

