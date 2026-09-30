# Развёртывание Notes через Ansible

inventory и playbook.


`ansible.cfg` ищет роль в соседнем каталоге через `roles_path = ../:./roles`.
Если разнесёте Git-репозитории по разным путям, измените `roles_path`
или установите роль в `./roles`.

## Что нужно заранее

- Контроллер: Linux, SSH-ключ для VM, Python с Ansible Core 2.20.
  Примеры ниже выполняются **на контроллере**, в `notes_deploy/`.
- Целевые хосты: чистые выделенные Ubuntu Server 22.04 с SSH и Python 3,
  пользователь с `sudo`, выход в Интернет к apt Docker, GitHub и реестру
  Docker Hub. Доступ по SSH должен оставаться разрешённым в сетевом
  firewall провайдера VM.
- Отредактируйте `inventory/hosts.yml`: IP `192.0.2.x` — примеры,
  не адреса работающих серверов. Группы `staging` и `prod` разделены.
- В inventory укажите реальный `ansible_user`. Если SSH использует
  нестандартный порт, укажите для хоста `ansible_port` и **то же значение**
  в `notes_ssh_port` до первого прогона. Иначе UFW может закрыть доступ.
  Если sudo запрашивает пароль, добавьте `--ask-become-pass` к командам запуска.

## Установить зависимости на контроллере

```bash
cd notes_deploy
python3 -m venv .venv
. .venv/bin/activate
python -m pip install 'ansible-core>=2.20,<2.21'
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook --version
```

Ansible использует модули `community.docker.docker_compose_v2` и
`community.general.ufw`. Docker SDK для Python на VM не требуется.
Установщик Docker в роли использует официальный apt-репозиторий и пакеты
`docker-ce`, `docker-ce-cli`, `containerd.io`,
`docker-buildx-plugin`, `docker-compose-plugin`.

## Задать секрет для каждой группы

Нужен пароль PostgreSQL длиной от 16 символов. Например, на контроллере
создайте случайное значение командой `openssl rand -hex 24`, затем поместите
его в отдельный **зашифрованный** Vault-файл каждой группы:

Если на VM **уже есть том PostgreSQL** от Блока 1, используйте в Vault его
существующий пароль: новый пароль в `.env` не изменит пароль внутри готовой БД.
Случайный новый пароль создавайте для чистой установки.

```bash
ansible-vault create inventory/group_vars/staging/vault.yml
ansible-vault create inventory/group_vars/prod/vault.yml
```

Содержимое каждого файла до шифрования:

```yaml
---
notes_postgres_password: "ВСТАВЬТЕ_СЮДА_СЛУЧАЙНОЕ_ЗНАЧЕНИЕ"
```

Путь `vault.yml` исключён через `.gitignore`, даже зашифрованные файлы
сюда коммитить не нужно. Не добавляйте пароль в `defaults/main.yml`,
`main.yml` группы, аргументы команды или `ansible.cfg`.
Для автоматизации вместо `--ask-vault-pass` используйте свою защищённую
систему выдачи секрета. На VM роль создаёт `/opt/test_app/.env` с правами
`0600`: пользователь с root-доступом или доступом к Docker может его увидеть.

## Запуск staging, затем prod

```bash
ansible-inventory --graph --ask-vault-pass
ansible staging -m ping --ask-vault-pass
ansible-playbook playbook.yml --syntax-check --ask-vault-pass
ansible-playbook playbook.yml --limit staging --ask-vault-pass
ansible-playbook verify.yml --limit staging --ask-vault-pass
```

Посмотрите результат, повторите playbook для проверки идемпотентности и
только затем запускайте production:

```bash
ansible-playbook playbook.yml --limit staging --ask-vault-pass
ansible-playbook playbook.yml --limit prod --ask-vault-pass
ansible-playbook verify.yml --limit prod --ask-vault-pass
```

При неизменной конфигурации второй прогон должен завершиться без изменений
состояния (если внешний Git ref не сдвинулся). Для production рекомендуется
зафиксировать `notes_app_git_version` на конкретном commit SHA в
`inventory/group_vars/prod/main.yml` после того, как вы определите нужный
commit. По умолчанию используется `master` upstream-репозитория;
приложение не редактируется, файлы контейнеризации поставляет роль.

Сервис слушает только `127.0.0.1:8000` **на самой VM**, поэтому проверка
по внешнему IP VM на порт 8000 не должна проходить. С контроллера используйте
SSH-туннель:

```bash
ssh -N -L 18000:127.0.0.1:8000 ubuntu@<IP_VM>
# В другом терминале:
curl -i http://127.0.0.1:18000/readyz
```

На VM через SSH:

```bash
sudo ufw status verbose
cd /opt/test_app
sudo docker compose -p test_app ps
sudo docker compose -p test_app logs --tail=100 app db
sudo docker inspect test_app-app-1 --format '{{ .Config.User }}'
```

Ожидается `readyz = HTTP 200`, оба контейнера `healthy`,
`Config.User = 10001:10001`. Имя проекта `test_app` намеренно совпадает
с ручным запуском из одноимённого каталога Блока 1: это позволяет повторно
использовать том `test_app_pgdata` при обновлении тех же сервисов.
**Не меняйте имя проекта, DB/USER или пароль на существующем томе**
без плана миграции. `docker system prune -f` в cron не удаляет volumes,
но может удалить остановленные контейнеры других проектов.

## Старые контейнеры и правила firewall

Контейнеры прежней версии того же Compose project обновляются при изменении
конфигурации; orphan-контейнеры проекта удаляются. Если старая установка
создала самостоятельный контейнер, мешающий запуску, перечислите только
подтверждённые имена в group_vars:

```yaml
notes_legacy_container_names:
  - old-notes-app
```

Роль удалит указанные контейнеры, сохранив их тома. Перед миграцией
существующей PostgreSQL сделайте резервную копию и проверьте имя тома.
Правило UFW открывает SSH, а приложение доступно только на loopback.
Если на VM уже есть **другие** контейнеры с опубликованными портами, Docker
может пропускать их трафик в обход UFW: проверьте `docker ps --format`
и настройте для них отдельные ограничения или перенесите проект на
выделенную VM.

## Проверка ограничений

В комплекте есть сценарий `verify.yml` для уже развернутого стека.
