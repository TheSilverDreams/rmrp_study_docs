# Развёртывание Материалов RMRP на Ubuntu (production)

Инструкция рассчитана на сервер Ubuntu c доступом по SSH и установленным Git. На машине уже могут работать другие веб-приложения — подразумевается использование Nginx как обратного прокси.

## 1. Подготовка сервера
```bash
sudo apt update
sudo apt install -y curl git build-essential nginx
```

Установите Node.js (LTS). Пример через `nvm`:
```bash
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
source ~/.nvm/nvm.sh
nvm install 18
nvm use 18
```

Установите менеджер процессов (pm2) глобально:
```bash
npm install -g pm2
```

Создайте рабочую директорию (пример):
```bash
sudo mkdir -p /var/www/rmrp
sudo chown $USER:$USER /var/www/rmrp
cd /var/www/rmrp
```

## 2. Клонирование репозитория
```bash
git clone https://github.com/<your-org>/rmrp.git
cd rmrp
```

## 3. Настройка окружения
### Backend
```bash
cd backend
cp .env.example .env   # при необходимости создайте .env вручную
npm install
npm run migrate
npm run seed            # заполнение стартовыми данными (опционально)
```

Рекомендуемые переменные `.env` (пример):
```
PORT=4000
JWT_SECRET=<случайная_строка>
NODE_ENV=production
```

### Frontend
```bash
cd ../frontend
npm install
npm run build            # создаёт dist/
```

## 4. Production-сборка
В корне проекта можно добавить общий скрипт:
```bash
npm install  # устанавливает зависимости для корневых dev-скриптов
npm run build  # у нас по умолчанию собирает фронтенд и backend (если настроить)
```

На практике:
1. Сборка фронта (`frontend/dist`).
2. Backend работает как API (порт 4000).
3. Nginx раздаёт статический фронтенд и проксирует API.

## 5. Запуск backend через PM2
```bash
cd /var/www/rmrp/backend
pm2 start npm --name rmrp-api -- run start
pm2 save      # сохранить конфигурацию процессов
```
Команда `npm run start` должна запускать production-сервер (убедитесь, что в package.json прописан корректный скрипт).

## 6. Настройка Nginx
Создайте файл `/etc/nginx/sites-available/rmrp-admin.ru`:
```nginx
server {
    listen 80;
    server_name rmrp-admin.ru;

    root /var/www/rmrp/frontend/dist;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:4000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Активация:
```bash
sudo ln -s /etc/nginx/sites-available/rmrp-admin.ru /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Если хотите HTTPS, используйте Certbot/Let’s Encrypt (пример):
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d rmrp-admin.ru
```

## 7. Проверка
- `pm2 status` — убедиться, что процесс работает.
- Открыть `http://rmrp-admin.ru`.

## 8. Обновление приложения
1. Зайти на сервер, перейти в репозиторий:
   ```bash
   cd /var/www/rmrp
   git fetch --all
   git checkout main
   git pull
   ```
2. Применить актуальные миграции (`npm run migrate` в backend) и при необходимости seed.
3. Пересобрать фронтенд:
   ```bash
   cd frontend
   npm install        # если изменились зависимости
   npm run build
   ```
4. Перезапустить backend:
   ```bash
   cd ../backend
   npm install        # при измененных зависимостях
   pm2 restart rmrp-api
   pm2 save
   ```
5. Очистить кэш Nginx (необязательно) и перезагрузить:
   ```bash
   sudo systemctl reload nginx
   ```

## 9. Дополнительные рекомендации
- Создайте системного пользователя (например, `rmrp`) и выполняйте деплой от него.
- Настройте резервное копирование `backend/src/db/database.sqlite` (cron + rclone/rsync).
- Логи: `pm2 logs rmrp-api`, `/var/log/nginx/access.log`, `/var/log/nginx/error.log`.
- Для автоматизации можно использовать GitHub Actions + deploy-скрипт (ssh+pull+build+restart).
- Если на сервере несколько приложений, следите за портами; backend RMRP использует 4000.

## 10. Быстрый чек-лист
- [ ] Git pull, npm install (backend + frontend)
- [ ] `npm run migrate`
- [ ] `npm run build` (frontend)
- [ ] `pm2 restart rmrp-api`
- [ ] `sudo systemctl reload nginx`
- [ ] Smoke-тест (зайти, выполнить вход, открыть материал и квиз)

После выполнения этих шагов приложение будет доступно на `https://rmrp-admin.ru` с маршрутизацией API через Nginx и фоновым запуском backend через pm2.
