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

# Задеплоить Traefik
docker stack deploy -c docker-compose.yaml traefik
```

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
├── docker-compose.yaml   # Основная конфигурация Docker Swarm
├── traefik.yml           # Статическая конфигурация Traefik
├── .env.example          # Пример переменных окружения
├── README.md             # Эта инструкция
├── acme/                 # Директория для Let's Encrypt (создается автоматически)
│   └── acme.json        # Хранилище сертификатов
└── logs/                 # Директория для логов (создается автоматически)
    ├── traefik.log      # Основные логи
    └── access.log       # Access логи
```

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
# Все логи
docker service logs traefik_traefik -f

# Только последние 100 строк
docker service logs traefik_traefik --tail 100

# Логи из файлов (JSON формат)
docker exec $(docker ps -q -f name=traefik) cat /var/log/traefik/traefik.log
docker exec $(docker ps -q -f name=traefik) cat /var/log/traefik/access.log
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
4. Проверьте права на acme.json: `ls -la acme/acme.json` (должно быть 600)

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
