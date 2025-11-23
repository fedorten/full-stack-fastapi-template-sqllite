<template>
  <div class="register-container">
    <div class="register-card">
      <h1>Регистрация</h1>
      <form @submit.prevent="handleRegister">
        <div class="form-group">
          <label>Email:</label>
          <input
            v-model="email"
            type="email"
            required
            placeholder="your@email.com"
          />
        </div>
        <div class="form-group">
          <label>Имя (необязательно):</label>
          <input
            v-model="fullName"
            type="text"
            placeholder="Ваше имя"
          />
        </div>
        <div class="form-group">
          <label>Пароль:</label>
          <input
            v-model="password"
            type="password"
            required
            minlength="8"
            placeholder="Минимум 8 символов"
          />
        </div>
        <div v-if="error" class="error">{{ error }}</div>
        <div v-if="success" class="success">{{ success }}</div>
        <button type="submit" :disabled="loading">
          {{ loading ? 'Регистрация...' : 'Зарегистрироваться' }}
        </button>
        <p class="login-link">
          Уже есть аккаунт? <router-link to="/login">Войти</router-link>
        </p>
      </form>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { authAPI } from '../services/api'

export default {
  name: 'Register',
  setup() {
    const router = useRouter()
    const email = ref('')
    const fullName = ref('')
    const password = ref('')
    const error = ref('')
    const success = ref('')
    const loading = ref(false)

    const handleRegister = async () => {
      error.value = ''
      success.value = ''
      loading.value = true
      
      try {
        await authAPI.register(email.value, password.value, fullName.value || null)
        // Автоматически логиним пользователя после регистрации
        try {
          const tokenData = await authAPI.login(email.value, password.value)
          localStorage.setItem('access_token', tokenData.access_token)
          success.value = 'Регистрация успешна! Перенаправление...'
          setTimeout(() => {
            router.push('/')
          }, 500)
        } catch (loginErr) {
          // Если автоматический логин не удался, перенаправляем на страницу входа
          error.value = 'Регистрация успешна, но не удалось войти. Пожалуйста, войдите вручную.'
          setTimeout(() => {
            router.push('/login')
          }, 2000)
        }
      } catch (err) {
        error.value = err.response?.data?.detail || 'Ошибка регистрации'
      } finally {
        loading.value = false
      }
    }

    return {
      email,
      fullName,
      password,
      error,
      success,
      loading,
      handleRegister,
    }
  },
}
</script>

<style scoped>
.register-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background: #f5f5f5;
}

.register-card {
  background: white;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 400px;
}

h1 {
  margin-bottom: 1.5rem;
  text-align: center;
  color: #333;
}

.form-group {
  margin-bottom: 1rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  color: #555;
  font-weight: 500;
}

input {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  box-sizing: border-box;
}

input:focus {
  outline: none;
  border-color: #4CAF50;
}

button {
  width: 100%;
  padding: 0.75rem;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  margin-top: 1rem;
}

button:hover:not(:disabled) {
  background: #45a049;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.error {
  color: #f44336;
  margin-top: 0.5rem;
  font-size: 0.9rem;
}

.success {
  color: #4CAF50;
  margin-top: 0.5rem;
  font-size: 0.9rem;
}

.login-link {
  text-align: center;
  margin-top: 1rem;
  color: #666;
}

.login-link a {
  color: #4CAF50;
  text-decoration: none;
}

.login-link a:hover {
  text-decoration: underline;
}
</style>

