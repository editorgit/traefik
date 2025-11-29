# Traefik v3.2 для Docker Swarm

Современная конфигурация Traefik v3.2 с автоматическим получением SSL сертификатов от Let's Encrypt и Prometheus метриками.

## Возможности

- **Traefik v3.2** - последняя версия
- **Let's Encrypt** - автоматическое получение и обновление SSL сертификатов
- **HTTP/3 (QUIC)** - поддержка современного протокола
- **Prometheus** - метрики на порту 8082
- **JSON логирование** - структурированные логи для ELK/Loki
- **Docker Swarm** - нативная интеграция с Swarm mode

## Быстрый старт

### Деплой

```bash
# Инициализировать Docker Swarm (если еще не сделано)
docker swarm init

# Создать overlay сеть для Traefik
docker network create --driver overlay --attachable proxy

# Задеплоить Traefik (можно с локальной машины)
docker stack deploy -c docker-compose.yaml traefik
```

**Примечание:** Docker volumes (`traefik-acme`, `traefik-logs`) создаются автоматически при первом деплое. Traefik сам создаст `acme.json` с правильными правами при запуске.

### Проверка

```bash
# Проверить статус сервиса
docker service ls | grep traefik

# Проверить логи
docker service logs traefik_traefik -f

# Проверить что сервис запущен
docker ps | grep traefik
```

## Структура проекта

```
traefik/
├── docker-compose.yaml   # Вся конфигурация (включая настройки Traefik)
└── README.md             # Эта инструкция
```

**Docker Volumes:**
- `traefik-acme` - Let's Encrypt сертификаты (acme.json)
- `traefik-logs` - Логи Traefik (traefik.log, access.log)

**Примечание:** Вся конфигурация Traefik находится в `docker-compose.yaml` в секции `command`. Отдельный файл `traefik.yml` не требуется.

## Конфигурация для приложений

Чтобы ваши приложения были доступны через Traefik, добавьте labels в `docker-compose.yaml`:

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - proxy
    deploy:
      labels:
        # Включить Traefik
        - "traefik.enable=true"

        # HTTP роутер с редиректом на HTTPS
        - "traefik.http.routers.myapp-http.rule=Host(`myapp.example.com`)"
        - "traefik.http.routers.myapp-http.entrypoints=http"
        - "traefik.http.routers.myapp-http.middlewares=https-redirect"

        # HTTPS роутер
        - "traefik.http.routers.myapp-https.rule=Host(`myapp.example.com`)"
        - "traefik.http.routers.myapp-https.entrypoints=https"
        - "traefik.http.routers.myapp-https.tls=true"
        - "traefik.http.routers.myapp-https.tls.certresolver=letsencrypt"

        # Сервис
        - "traefik.http.services.myapp.loadbalancer.server.port=8000"

        # HTTPS redirect middleware
        - "traefik.http.middlewares.https-redirect.redirectscheme.scheme=https"
        - "traefik.http.middlewares.https-redirect.redirectscheme.permanent=true"

networks:
  proxy:
    external: true
```

### С IP whitelist

Если нужно ограничить доступ по IP, передайте `ALLOWED_IPS` при деплое приложения:

```yaml
deploy:
  labels:
    - "traefik.http.routers.myapp-https.middlewares=myapp-ipwhitelist"
    - "traefik.http.middlewares.myapp-ipwhitelist.ipwhitelist.sourcerange=${ALLOWED_IPS}"
```

Затем деплой с переменной:

```bash
export ALLOWED_IPS="203.0.113.1,203.0.113.2"
docker stack deploy -c docker-compose.yaml myapp
```

## Prometheus метрики

Метрики доступны на порту `8082`:

```bash
# Проверить метрики
docker exec $(docker ps -q -f name=traefik) wget -qO- http://localhost:8082/metrics
```

Для подключения к Prometheus добавьте scrape config:

```yaml
scrape_configs:
  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8082']
```

## Управление

### Обновление Traefik

```bash
# Обновить образ
docker service update --image traefik:v3.2 traefik_traefik

# Или через stack deploy
docker stack deploy -c docker-compose.yaml traefik
```

### Просмотр логов

```bash
# Логи через Docker service
docker service logs traefik_traefik -f

# Только последние 100 строк
docker service logs traefik_traefik --tail 100

# Логи из Docker volume (JSON формат)
docker exec $(docker ps -q -f name=traefik) cat /var/log/traefik/traefik.log
docker exec $(docker ps -q -f name=traefik) cat /var/log/traefik/access.log
```

### Доступ к сертификатам

```bash
# Просмотр acme.json
docker run --rm -v traefik-acme:/acme alpine cat /acme/acme.json

# Копировать acme.json на хост
docker run --rm -v traefik-acme:/acme -v $(pwd):/backup alpine cp /acme/acme.json /backup/
```

## Безопасность

### TLS/SSL

- **TLS 1.3** включен по умолчанию
- **HTTP/3 (QUIC)** поддерживается
- Сертификаты обновляются автоматически каждые 90 дней
- HTTP автоматически редиректится на HTTPS

### IP Whitelist

IP whitelist настраивается на уровне приложений через их labels (см. раздел "С IP whitelist" выше).

## Совместимость

✅ Все существующие Traefik v2+ labels работают без изменений

Ваш `wizipro` proxy service не требует изменений - все labels совместимы.

## Troubleshooting

### Сертификаты не получаются

1. Проверьте что домен указывает на сервер: `dig your-domain.com`
2. Проверьте что порт 80 открыт (HTTP Challenge)
3. Проверьте логи: `docker service logs traefik_traefik | grep acme`
4. Проверьте что acme.json создан в volume:
   ```bash
   docker run --rm -v traefik-acme:/acme alpine ls -la /acme/acme.json
   ```

### Приложение не доступно через Traefik

1. Проверьте что приложение в сети `proxy`
2. Проверьте labels: `docker service inspect myapp`
3. Проверьте логи Traefik
4. Убедитесь что `traefik.enable=true` установлен

## Дополнительные ресурсы

- [Traefik v3 Documentation](https://doc.traefik.io/traefik/)
- [Migration Guide v2 → v3](https://doc.traefik.io/traefik/v3.0/migration/v2-to-v3/)
- [Docker Swarm Provider](https://doc.traefik.io/traefik/providers/docker/)
- [Let's Encrypt](https://letsencrypt.org/)
