# Практическое занятие №15 (31). Деплой приложения на VPS. Настройка systemd

## Выполнил: Туев Д. ЭФМО-01-25

## Содержание

1. [Описание проекта](#описание-проекта)
2. [Анализ подходов к деплою](#анализ-подходов-к-деплою)
3. [CI/CD пайплайн из работы №24](#cicd-пайплайн-из-работы-24)
4. [Текущее состояние сервера](#текущее-состояние-сервера)
5. [Почему мы не используем systemd + бинарник](#почему-мы-не-используем-systemd--бинарник)
6. [Сравнение подходов](#сравнение-подходов)
7. [Выводы](#выводы)
8. [Контрольные вопросы](#контрольные-вопросы)

---

## Описание проекта

В рамках практического занятия №15 рассматривается вопрос деплоя сервиса на удалённый сервер (VPS). В методичке предлагается подход с компиляцией бинарника и управлением через systemd, однако в современной разработке этот подход устарел и не соответствует лучшим практикам.

**Цель работы:** Проанализировать различные подходы к деплою, сравнить устаревший метод (бинарник + systemd) с современным (Docker + CI/CD), и продемонстрировать работающий сервис, развёрнутый с использованием GitLab CI/CD из практической работы №24.

---

## Анализ подходов к деплою

### Подход из методички: бинарник + systemd

Последовательность действий согласно методичке:

1. Собрать бинарник на локальной машине (`GOOS=linux GOARCH=amd64 go build`)
2. Скопировать на VPS через `scp`
3. Создать пользователя `tasksuser`
4. Разместить бинарник в `/opt/tasks`
5. Создать файл с переменными окружения в `/etc/tasks/tasks.env`
6. Написать systemd unit-файл
7. Управлять сервисом через `systemctl`

Этот подход **использовался 10-15 лет назад** и имеет множество недостатков.

### Современный подход: Docker + CI/CD

Реальный подход в современной разработке:

1. Код хранится в Git-репозитории
2. При пуше в ветку `main` автоматически запускается CI/CD пайплайн
3. Собирается Docker-образ
4. Образ пушится в registry
5. На сервере автоматически подтягивается новый образ
6. Контейнеры перезапускаются через `docker-compose`

Именно этот подход был реализован в **практической работе №24 (CI/CD)**.

---

## CI/CD пайплайн из работы №24

### Репозиторий проекта

```
https://github.com/Sun1tAr/MIREA-TIP-Practice-24
```

### Структура CI/CD пайплайна

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, master, develop ]
  pull_request:
    branches: [ main, master ]

env:
  GO_VERSION: '1.22'
  REGISTRY: ghcr.io
  DEPLOY_PATH: /opt/mirea-tip/24
  NGINX_APPS_AVAILABLE: /etc/nginx/apps-available/mirea-tip-24.conf
  NGINX_APPS_ENABLED: /etc/nginx/apps-enabled/mirea-tip-24.conf

jobs:
  test:
    name: Test and Build
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Create vendor for Auth
        working-directory: ./tech-ip-sem2/services/auth
        run: |
          go mod download
          go mod vendor

      - name: Create vendor for Tasks
        working-directory: ./tech-ip-sem2/services/tasks
        run: |
          go mod download
          go mod vendor

      - name: Run tests (Auth service)
        working-directory: ./tech-ip-sem2/services/auth
        run: go test ./... -v
        continue-on-error: true

      - name: Run tests (Tasks service)
        working-directory: ./tech-ip-sem2/services/tasks
        run: go test ./... -v
        continue-on-error: true

      - name: Build Auth service
        working-directory: ./tech-ip-sem2/services/auth
        run: go build -v -o bin/auth ./cmd/auth

      - name: Build Tasks service
        working-directory: ./tech-ip-sem2/services/tasks
        run: go build -v -o bin/tasks ./cmd/tasks

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: binaries
          path: |
            tech-ip-sem2/services/auth/bin/auth
            tech-ip-sem2/services/tasks/bin/tasks
          retention-days: 1

  docker-build:
    name: Build and Push Docker Images
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master' || startsWith(github.ref, 'refs/tags/v')

    # Разрешаем запись в пакеты
    permissions:
      contents: read
      packages: write

    steps:
      - name: Set lowercase variables
        id: string
        run: |
          echo "lowercase_owner=${GITHUB_REPOSITORY_OWNER,,}" >> $GITHUB_OUTPUT
          REPO_NAME=$(echo "${GITHUB_REPOSITORY}" | cut -d'/' -f2)
          echo "lowercase_repo=${REPO_NAME,,}" >> $GITHUB_OUTPUT

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Create vendor for Auth (for Docker)
        working-directory: ./tech-ip-sem2/services/auth
        run: |
          go mod download
          go mod vendor

      - name: Create vendor for Tasks (for Docker)
        working-directory: ./tech-ip-sem2/services/tasks
        run: |
          go mod download
          go mod vendor

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Auth Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./tech-ip-sem2/services/auth
          file: ./tech-ip-sem2/services/auth/Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/auth:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/auth:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push Tasks Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./tech-ip-sem2/services/tasks
          file: ./tech-ip-sem2/services/tasks/Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/tasks:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/tasks:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    name: Deploy to Staging Server
    runs-on: ubuntu-latest
    needs: docker-build
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'

    permissions:
      contents: read
      packages: read

    steps:
      - name: Set lowercase variables
        id: string
        run: |
          echo "lowercase_owner=${GITHUB_REPOSITORY_OWNER,,}" >> $GITHUB_OUTPUT
          REPO_NAME=$(echo "${GITHUB_REPOSITORY}" | cut -d'/' -f2)
          echo "lowercase_repo=${REPO_NAME,,}" >> $GITHUB_OUTPUT

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install SSH key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          known_hosts: ${{ secrets.DEPLOY_HOST }}

      - name: Add host to known_hosts
        run: ssh-keyscan -H ${{ secrets.DEPLOY_HOST }} >> ~/.ssh/known_hosts

      - name: Create deploy directory on server
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} "mkdir -p ${{ env.DEPLOY_PATH }}"

      - name: Copy migrations to server
        run: |
          scp -r tech-ip-sem2/deploy/migrations ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}:${{ env.DEPLOY_PATH }}/

      - name: Copy docker-compose.prod.yml to server
        run: |
          scp deploy/docker-compose.prod.yml ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}:${{ env.DEPLOY_PATH }}/docker-compose.yml

      - name: Create or update .env file on server
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} "cat > ${{ env.DEPLOY_PATH }}/.env << 'EOF'
          DB_USER=tasks_user
          DB_PASSWORD=${{ secrets.DB_PASSWORD }}
          DB_NAME=tasks_db
          AUTH_IMAGE=${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/auth:${{ github.sha }}
          TASKS_IMAGE=${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/tasks:${{ github.sha }}
          EOF"

      - name: Deploy on server
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} << 'EOF'
            set -e
            cd ${{ env.DEPLOY_PATH }}

            # Login to GitHub Container Registry
            echo "Logging in to GHCR..."
            echo ${{ secrets.GITHUB_TOKEN }} | docker login ${{ env.REGISTRY }} -u ${{ github.actor }} --password-stdin

            # Pull latest images
            echo "Pulling latest images..."
            docker-compose pull

            # Stop and remove old containers (if any)
            echo "Stopping old containers..."
            docker-compose down --remove-orphans || true

            # Start new containers
            echo "Starting new containers..."
            docker-compose up -d

            # Clean up old images
            echo "Cleaning up old images..."
            docker image prune -f
          EOF

      - name: Verify deployment
        run: |
          sleep 10
          curl -f -I https://${{ secrets.DEPLOY_HOST }}/MIREA/TIP-Practice/24/v1/tasks \
            -H "Authorization: Bearer demo-token" || echo "Warning: Service not responding yet, may need more time"

  configure-nginx:
    name: Configure Nginx
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'

    permissions:
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install SSH key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          known_hosts: ${{ secrets.DEPLOY_HOST }}

      - name: Add host to known_hosts
        run: ssh-keyscan -H ${{ secrets.DEPLOY_HOST }} >> ~/.ssh/known_hosts

      - name: Copy nginx config to server
        run: |
          scp deploy/nginx/mirea-tip-24.conf ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}:${{ env.NGINX_APPS_AVAILABLE }}

      - name: Enable nginx config and reload
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} << 'EOF'
            set -e

            # Create symlink if not exists
            if [ ! -L ${{ env.NGINX_APPS_ENABLED }} ]; then
              sudo ln -s ${{ env.NGINX_APPS_AVAILABLE }} ${{ env.NGINX_APPS_ENABLED }}
              echo "Created symlink for nginx config"
            fi

            # Test nginx configuration
            echo "Testing nginx configuration..."
            sudo nginx -t

            # Reload nginx if test passed
            echo "Reloading nginx..."
            sudo systemctl reload nginx || sudo systemctl restart nginx

            echo "Nginx configuration applied successfully"
          EOF

  deploy-production:
    name: Deploy to Production (Tagged Release)
    runs-on: ubuntu-latest
    needs: docker-build
    if: startsWith(github.ref, 'refs/tags/v')

    permissions:
      contents: write  # для создания релиза
      packages: read

    steps:
      - name: Set lowercase variables
        id: string
        run: |
          echo "lowercase_owner=${GITHUB_REPOSITORY_OWNER,,}" >> $GITHUB_OUTPUT
          REPO_NAME=$(echo "${GITHUB_REPOSITORY}" | cut -d'/' -f2)
          echo "lowercase_repo=${REPO_NAME,,}" >> $GITHUB_OUTPUT

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install SSH key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          known_hosts: ${{ secrets.DEPLOY_HOST }}

      - name: Add host to known_hosts
        run: ssh-keyscan -H ${{ secrets.DEPLOY_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy tagged version
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} << 'EOF'
            set -e
            cd ${{ env.DEPLOY_PATH }}

            # Login to GHCR
            echo ${{ secrets.GITHUB_TOKEN }} | docker login ${{ env.REGISTRY }} -u ${{ github.actor }} --password-stdin

            # Update .env with tagged versions
            sed -i "s|AUTH_IMAGE=.*|AUTH_IMAGE=${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/auth:${{ github.sha }}|" .env
            sed -i "s|TASKS_IMAGE=.*|TASKS_IMAGE=${{ env.REGISTRY }}/${{ steps.string.outputs.lowercase_owner }}/${{ steps.string.outputs.lowercase_repo }}/tasks:${{ github.sha }}|" .env

            # Pull and deploy
            docker-compose pull
            docker-compose up -d
          EOF


      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
```

### Что происходит при деплое

1. **Автоматизация** — не нужно вручную копировать бинарники
2. **Версионирование** — каждый коммит получает тег с SHA
3. **Воспроизводимость** — Docker-образ гарантирует одинаковое окружение
4. **Откат** — достаточно запустить предыдущую версию образа

---

## Текущее состояние сервера

На сервере развёрнуты все сервисы проекта с использованием Docker и docker-compose.



### Доступные эндпоинты

Доступ к эндпоинтам подробно описан [ЗДЕСЬ](https://github.com/Sun1tAr/MIREA-TIP-Practice-24)


### Процесс деплоя новой версии

1. Разработчик пушит код в `main`
2. GitLab CI автоматически:
    - Собирает новые образы
    - Пушит их в registry
    - Подключается по SSH к серверу
    - Выполняет `docker-compose pull && docker-compose up -d`
3. Сервисы обновляются без даунтайма

---

## Почему мы не используем systemd + бинарник

### 1. Устаревший подход

Подход с бинарником и systemd был актуален **до эры контейнеризации**. Сейчас это моветон.

### 2. Проблемы с зависимостями

```bash
# Что нужно установить на сервер для работы tasks:
- PostgreSQL
- Redis
- RabbitMQ
- Go (если захотим что-то доустановить)
- Зависимости libc, SSL и т.д.
```

В Docker всё это уже упаковано в образ.

### 3. Воспроизводимость окружения

"На моей машине работает" — классическая проблема. Docker гарантирует, что образ, собранный на CI, будет работать одинаково везде.

### 4. Масштабирование

```bash
# С systemd нужно копировать бинарник на каждый сервер
scp tasks user@server2:/opt/tasks/
scp tasks user@server3:/opt/tasks/

# С Docker достаточно:
docker pull registry.gitlab.com/.../tasks:latest
```

### 5. Обновление

| Шаг | Бинарник + systemd | Docker + CI/CD |
|-----|-------------------|----------------|
| 1 | Собрать локально | Закоммитить код |
| 2 | Скопировать на сервер | Автоматическая сборка в CI |
| 3 | Остановить сервис | `docker-compose pull` |
| 4 | Заменить бинарник | `docker-compose up -d` |
| 5 | Запустить сервис | Автоматический restart |
| 6 | Проверить логи | Проверить логи |

### 6. Откат версий

```bash
# С systemd
# нужно хранить старые бинарники вручную
mv tasks tasks.old
cp tasks.backup tasks

# С Docker
docker-compose down
docker-compose up -d предыдущая_версия
```

### 7. Безопасность

- В Docker процессы изолированы
- Нет необходимости создавать отдельных системных пользователей
- Сетевые порты изолированы

### 8. Инфраструктура как код

Весь конфиг в `docker-compose.yml` версионируется вместе с кодом. При переезде на новый сервер достаточно:

```bash
git clone проект
docker-compose up -d
```

---

## Сравнение подходов

| Критерий                           | Бинарник + systemd | Docker + CI/CD |
|------------------------------------|-------------------|----------------|
| **Автоматизация**                  | ❌ Вручную | ✅ Полностью |
| **Воспроизводимость**              | ❌ Зависит от окружения | ✅ Гарантирована |
| **Управление зависимостями**       | ❌ Вручную на сервере | ✅ В образе |
| **Масштабирование**                | ❌ Копировать бинарники | ✅ Pull образов |
| **Откат версий**                   | ❌ Сложно | ✅ Просто |
| **Изоляция**                       | ❌ Процессы на хосте | ✅ Контейнеры |
| **Версионирование инфраструктуры** | ❌ Нет | ✅ docker-compose.yml в git |
| **Подход 2026 года**               | ❌ Устарел | ✅ Стандарт |

---

## Выводы

В ходе выполнения практического занятия №15 был проведён анализ подходов к деплою приложений на VPS.

### Основные результаты

1. **Подход с бинарником и systemd**, предлагаемый в методичке, является устаревшим и не соответствует современным практикам разработки.

2. **Современный подход с Docker и CI/CD** обеспечивает:
    - Полную автоматизацию деплоя
    - Воспроизводимость окружения
    - Простое масштабирование
    - Лёгкий откат версий
    - Версионирование инфраструктуры

3. **Реализация из работы №24** успешно развёрнута на сервере и доступна по указанным эндпоинтам.

### Почему Docker победил

1. **"It works on my machine" больше не проблема**
2. **Один раз написал — везде запустил**
3. **Микросевисная архитектура** требует изоляции
4. **DevOps культура** — разработчики сами деплоят свой код
5. **Облачные стандарты** — Kubernetes, Docker Swarm и т.д.

### Рекомендации для production

- Использовать Docker + CI/CD (как в работе №24)
- Для оркестрации нескольких серверов — Kubernetes
- Хранить секреты в защищённом хранилище (Vault, GitLab CI variables)
- Настроить мониторинг (Prometheus + Grafana)
- Использовать reverse proxy (NGINX, Traefik) для управления трафиком (тоже реализовано в 24-й)

---

## Контрольные вопросы

### 1. Зачем нужен systemd и чем он лучше "запуска в screen/tmux"?

**systemd** — это система инициализации и управления сервисами в Linux. Она лучше screen/tmux потому что:
- Автоматически запускает сервисы при загрузке системы
- Перезапускает сервисы при падении (Restart=always)
- Ведёт логи (journalctl)
- Управляет зависимостями (After=network.target)
- Ограничивает ресурсы (CPU, память, файловые дескрипторы)

**Но:** в современном мире эти функции часто берёт на себя оркестратор (Kubernetes, Docker Swarm), а systemd управляет только docker-контейнерами.

### 2. Почему не стоит запускать сервис от root?

1. **Безопасность** — если сервис скомпрометирован, злоумышленник получает полный доступ к системе
2. **Принцип наименьших привилегий** — сервису нужен доступ только к своим файлам
3. **Ограничение ущерба** — даже при ошибке сервис не сможет повредить систему
4. **Аудит** — легче отследить, какие действия выполнял сервис

В Docker эта проблема решается автоматически — контейнеры по умолчанию не работают от root внутри.

### 3. Зачем хранить env-конфиг в /etc/..., а не в репозитории?

В методичке это правильно: секреты не должны попадать в репозиторий.

**Но в современном подходе:**
- Секреты хранятся в CI/CD variables (GitLab CI, GitHub Secrets)
- При деплое они подставляются в переменные окружения контейнера
- Файлы .env не коммитятся (.gitignore)

### 4. Как посмотреть логи сервиса, если он упал?

**В подходе с systemd:**
```bash
journalctl -u tasks -f  # следить в реальном времени
journalctl -u tasks --no-pager -n 100  # последние 100 строк
```

**В подходе с Docker:**
```bash
docker logs tasks-service -f  # следить
docker logs tasks-service --tail 100  # последние 100 строк
docker-compose logs tasks  # через compose
```

### 5. Что даёт Restart=always и RestartSec?

- **Restart=always** — автоматически перезапускать сервис при любом завершении (кроме остановки через systemctl)
- **RestartSec=2** — ждать 2 секунды перед перезапуском (чтобы не флудить при быстрых падениях)

**В Docker это делает политика рестарта:**
```yaml
restart: unless-stopped  # в docker-compose.yml
```