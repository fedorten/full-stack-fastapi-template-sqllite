<template>
  <div class="profile-container">
    <header class="header">
      <button @click="goBack" class="back-btn">← Назад</button>
      <h1>Профиль</h1>
      <div></div>
    </header>

    <div v-if="loading" class="loading">Загрузка...</div>
    <div v-else class="profile-content">
      <div class="profile-card">
        <h2>Информация о пользователе</h2>
        <div class="profile-info">
          <div class="info-item">
            <label>Email:</label>
            <span>{{ user.email }}</span>
          </div>
          <div class="info-item">
            <label>Имя:</label>
            <span>{{ user.full_name || 'Не указано' }}</span>
          </div>
          <div class="info-item">
            <label>ID:</label>
            <span>{{ user.id }}</span>
          </div>
        </div>

        <div class="danger-zone">
          <h3>Опасная зона</h3>
          <p>Удаление аккаунта необратимо. Все ваши данные будут удалены.</p>
          <button @click="handleDelete" class="delete-btn" :disabled="deleting">
            {{ deleting ? 'Удаление...' : 'Удалить аккаунт' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { authAPI } from '../services/api'

export default {
  name: 'Profile',
  setup() {
    const router = useRouter()
    const user = ref(null)
    const loading = ref(true)
    const deleting = ref(false)

    const loadUser = async () => {
      try {
        user.value = await authAPI.getCurrentUser()
      } catch (error) {
        console.error('Error loading user:', error)
        router.push('/login')
      } finally {
        loading.value = false
      }
    }

    const handleDelete = async () => {
      if (!confirm('Вы уверены, что хотите удалить свой аккаунт? Это действие необратимо.')) {
        return
      }

      deleting.value = true
      try {
        await authAPI.deleteAccount()
        localStorage.removeItem('access_token')
        alert('Аккаунт успешно удален')
        router.push('/login')
      } catch (error) {
        console.error('Error deleting account:', error)
        alert(error.response?.data?.detail || 'Ошибка удаления аккаунта')
      } finally {
        deleting.value = false
      }
    }

    const goBack = () => {
      router.push('/')
    }

    onMounted(() => {
      loadUser()
    })

    return {
      user,
      loading,
      deleting,
      handleDelete,
      goBack,
    }
  },
}
</script>

<style scoped>
.profile-container {
  max-width: 800px;
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

.header h1 {
  margin: 0;
  flex: 1;
  text-align: center;
}

.profile-content {
  background: white;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.profile-card h2 {
  margin-top: 0;
  color: #333;
}

.profile-info {
  margin-bottom: 2rem;
}

.info-item {
  display: flex;
  padding: 1rem;
  border-bottom: 1px solid #eee;
}

.info-item label {
  font-weight: 500;
  width: 150px;
  color: #666;
}

.info-item span {
  color: #333;
}

.danger-zone {
  margin-top: 2rem;
  padding: 1.5rem;
  background: #fff3f3;
  border-radius: 8px;
  border: 1px solid #ffcdd2;
}

.danger-zone h3 {
  margin-top: 0;
  color: #d32f2f;
}

.danger-zone p {
  color: #666;
  margin-bottom: 1rem;
}

.delete-btn {
  padding: 0.75rem 1.5rem;
  background: #f44336;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
}

.delete-btn:hover:not(:disabled) {
  background: #d32f2f;
}

.delete-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.loading {
  text-align: center;
  padding: 2rem;
  color: #666;
}
</style>

