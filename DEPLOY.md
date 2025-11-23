# Деплой на сервер paerser2.ru

## Подготовка сервера

1. Подключитесь к серверу:
```bash
ssh root@217.18.61.113
```

2. Установите Docker и Docker Compose (если еще не установлены):
```bash
# Установка Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Установка Docker Compose
apt-get update
apt-get install -y docker-compose-plugin
```

3. Установите Git (если еще не установлен):
```bash
apt-get install -y git
```

## Настройка DNS

Убедитесь, что домен `paerser2.ru` указывает на IP сервера `217.18.61.113`:
- A запись: `paerser2.ru` -> `217.18.61.113`
- A запись: `*.paerser2.ru` -> `217.18.61.113` (опционально, для поддоменов)

## Деплой приложения

1. На сервере создайте директорию для проекта:
```bash
mkdir -p /opt/messager
cd /opt/messager
```

2. Скопируйте файлы проекта на сервер (с вашего локального компьютера):
```bash
# С вашего локального компьютера
cd /home/fedor/Desktop/lern_projects/mihail_messager2/full-stack-fastapi-template
rsync -avz --exclude 'node_modules' --exclude '.venv' --exclude '__pycache__' --exclude '*.pyc' . root@217.18.61.113:/opt/messager/
```

Или используйте Git:
```bash
# На сервере
cd /opt/messager
git clone <your-repo-url> .
```

3. На сервере создайте файл `.env`:
```bash
cd /opt/messager
cp .env.example .env
nano .env
```

Заполните необходимые переменные:
```env
DOMAIN=paerser2.ru
ENVIRONMENT=production
DOCKER_IMAGE_BACKEND=messager-backend
DOCKER_IMAGE_FRONTEND=messager-frontend
TAG=latest
BACKEND_CORS_ORIGINS=https://paerser2.ru,http://paerser2.ru

# Сгенерируйте секретный ключ:
# python3 -c "import secrets; print(secrets.token_urlsafe(32))"
SECRET_KEY=<your-secret-key>

FIRST_SUPERUSER=admin@paerser2.ru
FIRST_SUPERUSER_PASSWORD=<strong-password>
SQLITE_DB_PATH=data/app.db
```

4. Сделайте скрипт деплоя исполняемым:
```bash
chmod +x deploy.sh
```

5. Запустите деплой:
```bash
./deploy.sh
```

## Настройка Nginx и SSL (если нужно)

Если вы хотите использовать Nginx как reverse proxy с SSL:

1. Установите Nginx и Certbot:
```bash
apt-get update
apt-get install -y nginx certbot python3-certbot-nginx
```

2. Создайте конфигурацию Nginx:
```bash
nano /etc/nginx/sites-available/paerser2.ru
```

Добавьте:
```nginx
server {
    listen 80;
    server_name paerser2.ru www.paerser2.ru;

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

3. Активируйте конфигурацию:
```bash
ln -s /etc/nginx/sites-available/paerser2.ru /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

4. Получите SSL сертификат:
```bash
certbot --nginx -d paerser2.ru -d www.paerser2.ru
```

## Управление приложением

### Просмотр логов:
```bash
cd /opt/messager
docker compose -f docker-compose.prod.yml logs -f
```

### Остановка:
```bash
docker compose -f docker-compose.prod.yml down
```

### Перезапуск:
```bash
docker compose -f docker-compose.prod.yml restart
```

### Обновление:
```bash
cd /opt/messager
git pull  # если используете Git
./deploy.sh
```

## Проверка работы

После деплоя проверьте:
- Frontend: https://paerser2.ru
- Backend API: https://paerser2.ru/api/v1/docs
- Health check: https://paerser2.ru/api/v1/utils/health-check/

## Troubleshooting

Если что-то не работает:

1. Проверьте логи:
```bash
docker compose -f docker-compose.prod.yml logs
```

2. Проверьте статус контейнеров:
```bash
docker compose -f docker-compose.prod.yml ps
```

3. Проверьте, что порты открыты:
```bash
netstat -tuln | grep -E ':(80|443)'
```

4. Проверьте DNS:
```bash
dig paerser2.ru
```

