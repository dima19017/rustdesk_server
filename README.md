# rustdesk_server

Ansible-роль для установки и настройки **RustDesk Server** — self-hosted серверной части RustDesk (hbbs + hbbr) — из официальных релизов GitHub.

## Описание

Роль полностью самостоятельна и выполняет весь цикл развертывания серверной части RustDesk:

- скачивает бинарники `hbbs`, `hbbr`, `rustdesk-utils` с официальных [releases GitHub](https://github.com/rustdesk/rustdesk-server/releases);
- устанавливает их в `/usr/local/bin`, данные хранит в `/var/lib/rustdesk-server`, версию — в маркере `/etc/rustdesk-server-version`;
- создает выделенного системного пользователя `rustdesk` и systemd-юниты `hbbs.service` + `hbbr.service`;
- открывает порты в UFW (TCP 21115/21116/21117, UDP 21116; при `rustdesk_server_web_client_enable: true` — дополнительно TCP 21118/21119);
- по завершении выводит параметры подключения для клиентов (адрес ID/relay-сервера и публичный ключ);
- поддерживает зафиксированную версию и бесшовное обновление до последней стабильной версии через GitHub API;
- полностью удаляет сервер и сопутствующие объекты при `rustdesk_server_enable: false`.

Структура задач соответствует корпоративному стандарту:

```
tasks/
├── main.yml
├── tasks_00_hostinit.yml   # преднастройка хоста: проверка ОС, unzip, пользователь, каталоги
├── tasks_01_preinit.yml    # определение версии и архитектуры, проверка необходимости установки
├── tasks_02_install.yml    # загрузка и установка бинарников
├── tasks_03_config.yml     # systemd-юниты, запуск сервисов, правила UFW
├── tasks_06_postinit.yml   # проверка сервисов, вывод параметров подключения
├── tasks_07_info.yml       # вывод параметров подключения (запуск по тегу)
└── tasks_99_remove.yml     # удаление
```

## Требования

- **Ansible** >= 2.12
- **Коллекция** `community.general` (модуль `community.general.ufw`). Установка:

  ```bash
  ansible-galaxy collection install community.general
  ```

  Либо через `requirements.yml`:

  ```yaml
  collections:
    - community.general
  ```

- **ОС**: Debian / Ubuntu (проверяется в `hostinit`: `ansible_os_family == "Debian"`)
- **Права**: root или sudo на целевых хостах
- **Пакеты** (ставятся ролью автоматически): `unzip`, `ufw` (при `rustdesk_server_firewall_enable: true`)

## Установка роли

Из git-репозитория через `ansible-galaxy`:

```bash
ansible-galaxy role install git+https://github.com/dima19017/rustdesk_server.git
```

Клонирование в каталог ролей:

```bash
git clone git@github.com:dima19017/rustdesk_server.git ~/.ansible/roles/rustdesk_server
```

Для использования внутри проекта — добавьте в `requirements.yml`:

```yaml
roles:
  - name: rustdesk_server
    src: git+https://github.com/dima19017/rustdesk_server.git
    version: main
```

и установите: `ansible-galaxy install -r requirements.yml`.

## Переменные

### Параметры роли

| Переменная | Тип | По умолчанию | Описание |
|---|---|---|---|
| `rustdesk_server_role_enable` | bool | — | **Обязательная.** `true` — задачи роли будут обработаны, `false` — переменные переданы в скоуп, задачи не выполняются |
| `rustdesk_server_role_update_enable` | bool | `false` | `true` — всегда обновляться до последней стабильной версии с GitHub API; `false` — использовать `rustdesk_server_version` |
| `rustdesk_server_role_debug_enable` | bool | `{{ ansible_check_mode }}` | `true` — вывод отладочной информации в stdout |
| `rustdesk_server_role_hosts` | list | `{{ ansible_play_hosts }}` | Хосты (или группа инвентаря), на которых будет выполнена роль |

### Параметры установки

| Переменная | Тип | По умолчанию | Описание |
|---|---|---|---|
| `rustdesk_server_enable` | bool | `true` | `true` — установить и настроить сервис; `false` — удалить |
| `rustdesk_server_version` | str | `"1.1.16"` | Версия сервера при `rustdesk_server_role_update_enable: false` |
| `rustdesk_server_public_host` | str | `{{ ansible_host }}` | Публичный IP/DNS сервера hbbs, который получат клиенты |
| `rustdesk_server_relay_host` | str | `{{ rustdesk_server_public_host }}` | Адрес relay-сервера (hbbr); укажите отдельно, если hbbr на другом хосте |
| `rustdesk_server_key` | str | `""` | Предзаданный ключ сервера (base64 публичный ключ). Пусто — сервер сгенерирует ключи сам при первом запуске |
| `rustdesk_server_user` | str | `"rustdesk"` | Системный пользователь, от имени которого запускаются hbbs/hbbr |
| `rustdesk_server_firewall_enable` | bool | `true` | `true` — роль добавит правила UFW для портов RustDesk |
| `rustdesk_server_web_client_enable` | bool | `false` | `true` — открыть порты веб-клиента (21118/21119 TCP) |

### Параметры портов

| Переменная | Тип | По умолчанию | Описание |
|---|---|---|---|
| `rustdesk_server_hbbs_nat_port` | int | `21115` | TCP-порт проверки NAT (hbbs) |
| `rustdesk_server_hbbs_port` | int | `21116` | TCP/UDP порт rendezvous/ID-сервера (hbbs) |
| `rustdesk_server_hbbr_port` | int | `21117` | TCP-порт relay-сервера (hbbr) |
| `rustdesk_server_hbbs_web_port` | int | `21118` | TCP-порт веб-клиента hbbs (при `web_client_enable: true`) |
| `rustdesk_server_hbbr_web_port` | int | `21119` | TCP-порт веб-клиента hbbr (при `web_client_enable: true`) |
| `rustdesk_server_firewall_tcp_ports` | list | авто | TCP-порты для UFW (формируются автоматически из портов выше) |
| `rustdesk_server_firewall_udp_ports` | list | авто | UDP-порты для UFW (порт rendezvous hbbs) |

### Константы (не переопределять)

Определены в `vars/main.yml`, при необходимости переопределяются только осознанно:

| Переменная | Значение | Назначение |
|---|---|---|
| `rustdesk_server_install_dir` | `/usr/local/bin` | Каталог установки бинарников |
| `rustdesk_server_data_dir` | `/var/lib/rustdesk-server` | Данные сервера; здесь hbbs хранит ключи `id_ed25519` и `id_ed25519.pub` |
| `rustdesk_server_log_dir` | `/var/log/rustdesk-server` | Каталог логов |
| `rustdesk_server_version_file` | `/etc/rustdesk-server-version` | Маркер установленной версии |
| `rustdesk_server_service_hbbs` / `rustdesk_server_service_hbbr` | `hbbs` / `hbbr` | Имена systemd-юнитов |
| `rustdesk_server_binaries` | `hbbs`, `hbbr`, `rustdesk-utils` | Список бинарников в архиве |
| `rustdesk_server_arch_map` | `x86_64→amd64`, `aarch64→arm64v8`, `armv7l→armv7` | Маппинг архитектуры на суффикс архива |
| `rustdesk_server_download_base` | `https://github.com/rustdesk/rustdesk-server/releases/download` | Базовый URL загрузки |
| `rustdesk_server_github_api_latest` | `https://api.github.com/repos/rustdesk/rustdesk-server/releases/latest` | URL API последней версии |

## Теги

| Тег | Задачи |
|---|---|
| `hostinit` | Преднастройка хоста (проверка ОС, установка `unzip`/`ufw`, создание пользователя и каталогов) |
| `preinit` | Определение версии, архитектуры, необходимости установки |
| `install` | Скачивание и установка бинарников, запись маркера версии |
| `config` | `preinit` + `config` + `postinit` (юниты, сервисы, UFW, проверка и вывод параметров) |
| `postinit` | Проверка сервисов, чтение ключа, вывод параметров подключения |
| `info` | Вывод параметров подключения для клиента |
| `remove` | Полное удаление сервера |
| `rustdesk_server` | Все задачи роли |
| `rustdesk_server_hostinit` | Только преднастройка хоста |
| `rustdesk_server_install` | Только установка |
| `rustdesk_server_config` | `preinit` + `config` + `postinit` (все, что связано с конфигурацией) |
| `rustdesk_server_info` | Только вывод параметров подключения |
| `rustdesk_server_remove` | Только удаление |

Примеры запуска:

```bash
# установка с выводом параметров для клиента
ansible-playbook -i inventory.yml playbook.yml --tags rustdesk_server_install,rustdesk_server_config

# только вывод параметров подключения для клиента (сервер уже установлен)
ansible-playbook -i inventory.yml playbook.yml --tags rustdesk_server_info

# полное удаление
ansible-playbook -i inventory.yml playbook.yml --tags rustdesk_server_remove
```

## Пример плейбука

```yaml
---
- name: "Deploy RustDesk Server"
  hosts: rustdesk
  gather_facts: true
  become: true

  vars:
    # Использование роли (обязательно)
    rustdesk_server_role_enable: true
    # Установка, а не удаление
    rustdesk_server_enable: true
    # Версия (используется при role_update_enable: false)
    rustdesk_server_version: "1.1.16"
    # Публичный адрес для клиентов
    rustdesk_server_public_host: "rustdesk.example.com"
    # Открыть порты веб-клиента 21118/21119
    rustdesk_server_web_client_enable: false

  tasks:
    - name: "Include rustdesk_server role"
      ansible.builtin.include_role:
        name: rustdesk_server
```

## Параметры подключения для клиента

После установки роль выводит параметры подключения (блок `postinit`):

```
RustDesk Server ready! Client connection parameters:
------------------------------------------------------
ID/Relay server (hbbs): rustdesk.example.com:21116
Relay server (hbbr):    rustdesk.example.com:21117
Public key:             <base64-ключ из id_ed25519.pub>
------------------------------------------------------
```

Получить их повторно (без переустановки):

```bash
ansible-playbook -i inventory.yml playbook.yml --tags rustdesk_server_info
```

Что ввести в клиенте RustDesk (**Settings → Network**):

| Поле клиента | Значение |
|---|---|
| ID/Relay Server | `rustdesk.example.com` (или `rustdesk.example.com:21116`) |
| Key | публичный ключ из вывода (`id_ed25519.pub`) |

> **Важно.** Ключи генерируются hbbs при первом запуске в `/var/lib/rustdesk-server` (`id_ed25519` + `id_ed25519.pub`) и **не резервируются** (осознанное решение). Потеря данных каталога приведет к смене ключа — клиенты придется перенастроить. Чтобы сохранить идентичность сервера при переустановке, задайте `rustdesk_server_key` (base64 публичного ключа).

## Обновление версии

Роль поддерживает два режима:

1. **Зафиксированная версия** (`rustdesk_server_role_update_enable: false`, по умолчанию) — устанавливается `rustdesk_server_version`.
2. **Последняя стабильная версия** (`rustdesk_server_role_update_enable: true`) — версия берется с GitHub API (`/repos/rustdesk/rustdesk-server/releases/latest`, поле `tag_name`).

Логика обновления:

- установленная версия читается из маркера `/etc/rustdesk-server-version`;
- если целевая версия отличается или отсутствует один из бинарников — роль скачивает новую версию, заменяет бинарники, и handler перезапускает сервисы `hbbs`/`hbbr`;
- если версия совпадает — установка пропускается (роль идемпотентна).

```yaml
# всегда держать сервер на последней стабильной версии
vars:
  rustdesk_server_role_enable: true
  rustdesk_server_role_update_enable: true
```

> Для запросов к GitHub API требуется доступ с хоста управления (ansible control node) к `api.github.com`. Если доступа нет — используйте зафиксированную версию.

## Удаление

Задайте `rustdesk_server_enable: false` — роль выполнит блок `remove`:

```yaml
vars:
  rustdesk_server_role_enable: true
  rustdesk_server_enable: false
```

Будут удалены: сервисы `hbbs`/`hbbr` (остановлены, отключены, юниты удалены), бинарники из `/usr/local/bin`, маркер версии, каталоги данных и логов, системный пользователь `rustdesk`, правила UFW.

> **Внимание:** удаляются также данные `/var/lib/rustdesk-server`, включая ключи сервера.

## Тестирование

Тесты на базе [molecule](https://ansible.readthedocs.io/projects/molecule/) (scenario `default`, driver `docker`):

- **Платформы**: Debian 12 (`geerlingguy/docker-debian12-ansible`), Ubuntu 24.04 (`geerlingguy/docker-ubuntu2404-ansible`);
- **Проверки** (`verify.yml`): наличие бинарников, активность сервисов `hbbs`/`hbbr`, наличие `id_ed25519.pub` и маркера версии, слушающие порты 21116/21117;
- **Идемпотентность**: в `test_sequence` включен шаг `idempotence`.

Запуск:

```bash
cd /Users/surtupindmitrij/dima/roles/rustdesk_server
molecule test
```

Локальный lint:

```bash
ansible-lint
yamllint .
```

## Портирование и совместимость

Поддерживаемые платформы (из `meta/main.yml`):

| ОС | Версии | Codename |
|---|---|---|
| Debian | 11, 12, 13 | bullseye, bookworm, trixie |
| Ubuntu | 20.04, 22.04, 24.04 | focal, jammy, noble |

Автоматическими тестами molecule покрыты **Debian 12** и **Ubuntu 24.04**.

Поддерживаемые архитектуры: `x86_64`/`amd64`, `aarch64`/`arm64` (`arm64v8`), `armv7l` (`armv7`).

Роль написана в корпоративном стиле (`tasks_XX_*.yml`, `argument_specs.yml`, role-скоп теги), поэтому перенос в другие роли репозитория не требуется — достаточно подключить её через `include_role` или в `roles` плейбука. Валидация переменных выполняется через `meta/argument_specs.yml` при использовании `ansible-playbook --syntax-check`/ansible-lint.
