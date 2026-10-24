# Outline + draw.io + Grist (Ansible + Docker Compose)

Вики [Outline](https://www.getoutline.com/), редактор диаграмм [draw.io](https://www.drawio.com/)
и таблицы [Grist](https://www.getgrist.com/) на двух узлах, разворачиваются одним плейбуком.

- Все сервисы в одном compose-проекте, образы `latest`
- Вход в Outline и Grist через Keycloak (OIDC)
- Внешние PostgreSQL и Redis
- Reverse proxy с vIP на тех же хостах, TLS терминируется на прокси
- Firewall на nftables: опубликованные порты доступны только прокси и мониторингу

## Структура

```
install_outline.yaml           # плейбук: firewall + деплой + rolling restart + очистка
inventory.example.yml          # пример inventory (скопировать в inventory.yml)
files/outline/
  outline.yaml                 # docker-compose
  outline.nft.j2               # шаблон правил nftables
  docker.env.example           # шаблон настроек Outline
  grist.env.example            # шаблон настроек Grist
  docker.env                   # НЕ в репозитории — реальные настройки Outline
  grist.env                    # НЕ в репозитории — реальные настройки Grist
```

## Требования

- Хосты Debian/Ubuntu с Docker Engine и плагином `docker compose`
- Ansible на управляющей машине (только встроенные модули)
- PostgreSQL с базами `outline` и `grist`, Redis
- Клиенты `wiki` и `grist` в реалме Keycloak
- **Общее хранилище** для `/outline/storage` и `/grist` на обоих узлах (см. ниже)

## Настройка

1. Inventory:
   ```bash
   cp inventory.example.yml inventory.yml
   ```
   Для каждого хоста — `ansible_host` и `node` (1 или 2, порядок перезапуска). Группа — `docker`.

2. Секреты:
   ```bash
   cp files/outline/docker.env.example files/outline/docker.env
   cp files/outline/grist.env.example  files/outline/grist.env
   ```
   Замените все `CHANGE_ME`. Новые ключи генерируются `openssl rand -hex 32`.
   **`SECRET_KEY` на работающей инсталляции не меняйте**: им зашифрованы данные в БД Outline.

   Можно дополнительно зашифровать оба файла: `ansible-vault encrypt files/outline/*.env` —
   модуль `copy` расшифрует их при копировании, плейбук запускайте с `--ask-vault-pass`.

3. Переменные плейбука (`install_outline.yaml`, блок `vars`):

   | Переменная | Назначение |
   |---|---|
   | `proxies` | IP reverse proxy, включая vIP |
   | `monitoring` | IP системы мониторинга |
   | `published_ports` | порты, которые закрывает firewall (совпадают с `ports:` в compose) |
   | `health_urls` | что проверяется после перезапуска узла |

## Развертывание

```bash
ansible-playbook -i inventory.yml install_outline.yaml
```

Плейбук:
1. Ставит nftables и применяет `/etc/nftables.d/outline.nft` (с проверкой синтаксиса).
2. Копирует compose и env-файлы (`0600`) в `/docker-compose/outline`.
3. Скачивает свежие образы.
4. Перезапускает **узел 1** и ждет, пока все три сервиса ответят 200, затем то же для **узла 2**.
5. Чистит Docker: `system prune -a` и `volume prune -a`.

> ⚠️ `docker volume prune -a` удаляет **все** тома, не используемые запущенными контейнерами, включая именованные.

## Сеть и firewall

| Порт | Сервис | Кому открыт |
|---|---|---|
| 4000 | Outline | `proxies`, Docker-сети `172.16.0.0/12`, `monitoring` |
| 8484 | Grist | то же |
| 8485 | draw.io | то же |

Порты, опубликованные Docker, не проходят через обычную цепочку `input`: Docker перенаправляет их
в контейнер (DNAT) раньше. Поэтому правила стоят в `prerouting` с приоритетом `-150` — до Docker,
пока виден настоящий адрес клиента. `172.16.0.0/12` нужна для прокси, работающего в контейнере
на том же хосте. Обращения с самого хоста (`curl 127.0.0.1:4000`, проверки плейбука) идут через
интерфейс `lo` и тоже проходят `prerouting`, поэтому `lo` разрешен отдельным правилом.

**Не делайте `systemctl reload/restart nftables`** на работающем хосте: стандартный `/etc/nftables.conf`
начинается с `flush ruleset` и снесет правила Docker. Применять правила — `nft -f /etc/nftables.d/outline.nft`.

## Хранилище и отказоустойчивость

`/outline/storage` и `/grist` — общие каталоги на NFS, смонтированные на обоих узлах.
Балансировка — nginx на тех же хостах.

- **Outline** — active-active (`ip_hash`). Состояние живет в PostgreSQL, Redis и общем каталоге вложений,
  поэтому экземпляры на разных узлах взаимозаменяемы.
- **Grist** — active-passive (`backup` в upstream). Документы Grist — файлы SQLite. При active-active
  два пользователя с разных IP могут открыть один документ на разных узлах, и два процесса начнут
  писать в один файл через NFS, где блокировки SQLite ненадежны, — это путь к поврежденному документу.
  `ip_hash` закрепляет клиента, а не документ, поэтому не помогает. Второй узел — горячий резерв:
  контейнер запущен, nginx переключается на него сам, если основной не отвечает.

```nginx
upstream wikiexmpleorg {
    ip_hash;
    server 11.11.11.11:4000 max_fails=2 fail_timeout=10s;
    server 22.22.22.22:4000 max_fails=2 fail_timeout=10s;
}

upstream grisexampleorg {
    server 11.11.11.11:8484 max_fails=2 fail_timeout=10s;
    server 22.22.22.22:8484 backup;
}
```

Для настоящего active-active у Grist нужно внешнее S3-хранилище документов (например, MinIO).
- Redis и PostgreSQL — по одному экземпляру: при их падении не работают оба узла.

## Проверка

```bash
docker ps                                           # outline и grist: healthy
curl -s http://127.0.0.1:4000/_health               # OK
curl -s http://127.0.0.1:8484/status                # Grist server ... is alive
nft list table inet outline
```

## Диагностика

**Страницы грузятся очень долго или не открываются** — скорее всего, firewall режет прокси.
Проверка: `nft delete table inet outline` (вернуть: `nft -f /etc/nftables.d/outline.nft`). Посмотреть, кого режут:
```bash
nft -a list chain inet outline prerouting           # handle строки с drop
nft insert rule inet outline prerouting position <HANDLE> tcp dport { 4000, 8484, 8485 } log prefix \"outline-drop: \"
journalctl -k -f | grep outline-drop                # SRC= — кого добавить в разрешенные
```
