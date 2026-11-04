# Portainer CE

Два варианта установки: обычный Docker Compose или Swarm stack с агентом на каждом Linux-узле. В обоих вариантах интерфейс открывается по `https://PORTAINER_DOMAIN` через Traefik с сертификатом Let's Encrypt. Порты Portainer 9000/9443/8000 и Agent 9001 на хост не публикуются; HTTP 9000 используется внутри Docker-сети за TLS-прокси.

## Docker Compose

Сначала запустите [Traefik](../traefik/README.md), создайте DNS-запись для Portainer. Из корня репозитория:

```bash
cd portainer
cp .env.example .env
# Укажите реальный PORTAINER_DOMAIN в .env.
docker compose config --quiet
docker compose up -d
docker compose logs --tail=100 portainer
```

Откройте свой `https://portainer.example.com`, выполните начальную настройку и создайте администратора. Если версия Portainer запрашивает setup token, возьмите его из логов запуска. Если истёк срок начальной настройки, перезапустите `docker compose restart portainer` и откройте страницу снова.

Portainer подключён к локальному Docker через socket. Если этот Docker уже является Swarm manager, локальное окружение позволяет управлять его Swarm; для установки агентов на все узлы используйте следующий вариант.

## Docker Swarm

### 1. Подготовить узлы

На каждом Linux-сервере должен работать Docker Engine. Команды менеджмента кластера выполняйте на manager. Для первого запуска достаточно одного manager; для отказоустойчивого quorum обычно используют три manager.

Разрешите между доверенными IP узлов:

| Порт | Назначение |
| --- | --- |
| TCP 2377 | Управление Swarm, доступ к manager |
| TCP/UDP 7946 | Обнаружение узлов |
| UDP 4789 | Трафик overlay-сетей |
| TCP 9001 | Связь Portainer с агентами по overlay-сети |

Порты Swarm и агентов не открывайте всему интернету. Публичные TCP 80/443 нужны на manager с Traefik. Агент 9001 работает внутри overlay, отдельный publish на хосте не нужен. Конфигурация использует стандартный Docker data-root `/var/lib/docker`; при его изменении исправьте bind mount volumes в `stack.yml`.

### 2. Инициализировать Swarm

На первом manager (замените IP на постоянный адрес его интерфейса, доступный остальным узлам):

```bash
docker swarm init --advertise-addr 10.0.0.10
docker node ls
docker swarm join-token worker
```

Вывод последней команды содержит `docker swarm join --token ... 10.0.0.10:2377`. Выполните эту команду на каждом worker. Для добавления manager получите отдельную команду через `docker swarm join-token manager`. Токены не сохраняйте в репозиторий.

На manager проверьте `docker node ls`: узлы должны быть `Ready` и `Active`. Если Swarm уже работает, повторный `swarm init` не нужен.

### 3. Подготовить общую сеть и запустить Traefik

Для новой установки из корня репозитория:

```bash
docker network create --driver overlay --attachable proxy
cd traefik
cp .env.example .env
# Заполните ACME_EMAIL в .env.
docker compose -f compose.yml -f compose.swarm.yml up -d
cd ../portainer
```

Если `.env` Traefik уже существует, используйте его, не перезаписывайте. Существующая сеть `proxy` должна иметь `Driver=overlay`, `Scope=swarm`, `Attachable=true`:

```bash
docker network inspect proxy --format '{{.Driver}} {{.Scope}} {{.Attachable}}'
```

**При переходе с обычного Compose** сначала остановите Portainer (`docker compose down` в `portainer`) и Traefik (`docker compose down` в `traefik`) без `-v`. Остановите или отключите от `proxy` остальные приложения. Проверьте подключённые контейнеры через `docker network inspect proxy`; только когда bridge-сеть пуста, удалите её через `docker network rm proxy`, создайте overlay-сеть командой выше и запустите Traefik с обоими файлами. Затем разверните Portainer stack и перезапустите приложения с новой сетью. На время перехода будет перерыв в работе.

### 4. Развернуть Portainer и агенты

Из каталога `portainer`, на том же manager, где должны храниться данные:

```bash
cp .env.example .env
# Заполните PORTAINER_DOMAIN. Если .env уже есть, сохраните его.
set -a
. ./.env
set +a

docker node update --label-add portainer.data=true "$(docker info --format '{{.Swarm.NodeID}}')"
docker stack config -c stack.yml
docker stack deploy -c stack.yml portainer
docker stack services portainer
docker service ps portainer_portainer
docker service ps portainer_agent
```

`docker stack deploy` не загружает `.env` автоматически, поэтому переменные экспортируются в shell до вызова. Загружайте только собственный доверенный `.env`. Через Portainer Web editor переменные нужно указать в настройках stack явно.

Portainer Server запускается в одной реплике на manager с меткой `portainer.data=true`. Назначайте эту метку только одному узлу: volume `portainer_data` локальный, перенос на другой узел без восстановления данных создаст пустую установку. Агент работает в `global`-режиме на каждом Linux-узле, включая новые присоединённые узлы. Server подключается к агентам через `tasks.agent:9001`; это окружение представляет весь Swarm, добавлять каждый узел вручную не требуется. `--tlsskipverify` применяется к внутреннему соединению с самоподписанными сертификатами агентов; браузер получает публичный сертификат от Traefik.

При переходе с Compose на stack на том же manager используется существующий volume `portainer_data`. Сделайте резервную копию перед переходом; одновременно подключать к нему два Portainer Server нельзя.

### 5. Проверить, что Portainer видит Swarm

Откройте домен Portainer, пройдите первичную настройку. Setup token, если он требуется вашей версией, доступен через:

```bash
docker service logs --tail=100 portainer_portainer
```

В списке **Environments** откройте преднастроенное окружение агента (название может быть `primary` или `local`). Должны быть доступны **Swarm**, **Nodes**, **Services** и **Stacks**; список Nodes должен совпадать с `docker node ls`. `portainer_agent` должен иметь по одной работающей задаче на Linux-узел. Для проверки публикации Swarm-сервиса используйте [пример Traefik](../traefik/README.md#сервисы-swarm).

Если окружение недоступно, проверьте `docker service ps portainer_agent`, логи сервера/агента, межузловые порты и overlay-сеть `portainer_agent_network`. При ошибке HTTPS смотрите также логи Traefik. Если просрочена первичная настройка, перезапустите сервер через `docker service update --force portainer_portainer`.

## Данные и обновление

Данные UI, настройки и пользователи хранятся в volume `portainer_data`. Docker socket и Agent дают административный доступ к узлам: ограничьте доступ к UI и используйте сильный пароль администратора. Не включайте `Force HTTPS only` внутри Portainer: в этой схеме TLS завершается на Traefik, а upstream использует HTTP.

Образы используют официальный канал `lts`. Server и Agent должны иметь согласованные версии; для воспроизводимого обновления задавайте одну и ту же проверенную версию обоих образов в `.env`. Перед обновлением сохраните резервную копию Portainer через UI либо остановите Server и сохраните его volume.

Для Compose: `docker compose pull` и `docker compose up -d`. Для Swarm: экспортируйте `.env` как выше, выполните `docker stack deploy --resolve-image always -c stack.yml portainer`, затем проверьте задачи через `docker stack services portainer`. Не удаляйте volume при остановке. Привязка к одному manager защищает от случайного запуска с пустыми данными, но не обеспечивает автоматическую отказоустойчивость Portainer.

## Официальная документация

- [Создание Swarm и межузловые порты](https://docs.docker.com/engine/swarm/swarm-tutorial/)
- [Установка Portainer CE в Swarm](https://docs.portainer.io/start/install-ce/server/swarm/linux)
- [Официальная схема Agent + Server](https://downloads.portainer.io/ce-lts/portainer-agent-stack.yml)
- [CLI Portainer](https://docs.portainer.io/advanced/cli)

