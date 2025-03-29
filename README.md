# Ansible Role: Users

## Описание
Роль для управления пользователями и группами на чистой Ubuntu Server 22 LTS. Реализует следующие возможности:
- Установка зависимостей (curl, cron, PowerShell).
- Создание групп пользователей.
- Управление пользователями (создание/удаление, настройка групп, установка оболочки, установка пароля).
- Добавление SSH-ключей через URL.
- Отключение пользователей по дате с помощью cron и PowerShell-скрипта.

Роль использует теги для гибкого управления задачами: setup, groups, users, expiry, ssh.

## Требования
- Ansible 2.9 или выше.
- Ubuntu Server 22 LTS.
- Доступ по SSH с правами `sudo`.

## Установка и запуск

### 1. Клонирование репозитория
Склонируйте проект с GitHub:
```
git clone https://github.com/shmakus/ansible-role-manage-users.git
cd ansible-role-manage-users
```

### 2. Создание файла секретов
Все пароли хранятся в зашифрованном файле `vault.yml`. Создайте его с помощью Ansible Vault:
```
ansible-vault create vault.yml
```
Введите пароль (например, `pass`) и добавьте содержимое в формате:
```
vault_global_admin_password: "{{ 'adminpass' | password_hash('sha512') }}"
vault_dev1_password: "{{ 'devpass' | password_hash('sha512') }}"
vault_local_user_password: "{{ 'localpass' | password_hash('sha512') }}"
vault_default_password: "{{ 'localpassdefault' | password_hash('sha512') }}"
```
**Примечание:** В текущем проекте vault.yml уже создан и зашифрован. Пароль доступа указан в файле vault_pass.txt, в секрет уже добавлены тестовые пароли, посмотреть можно командой.
```
ansible-vault edit vault.yml --vault-password-file vault_pass.txt
```

### 3. Создание файла пароля Vault
Сохраните пароль Vault в файл:
```
echo "pass" > vault_pass.txt
chmod 600 vault_pass.txt
```
**Примечание:** Файл `vault_pass.txt` по правильному не должен добавляеться в Git, но в качестве тестового задания данный файл был намеренно не добавлен в `.gitignore`.

### 4. Формирование файла `inventory/hosts.yml`
### Структура файла
Файл разделён на несколько уровней:
```
all: Глобальные настройки и пользователи, применяемые ко всем хостам.
children: Подгруппы, такие как dev_servers, с дополнительными настройками.
hosts: Конкретные хосты (например, dev_host1) с локальными переменными
```

### Переменные
Общие параметры подключения:
```
ansible_connection: ssh — метод подключения.
ansible_user: root — пользователь для подключения.
ansible_password: password — пароль для подключения (для тестового окружения).
ansible_port: 2222 — порт SSH.
ansible_become: yes — включение повышения привилегий (sudo).
```
`default_groups`: Список групп, которые создаются на хостах (например, admins, developers).

`users`: Список пользователей с параметрами:
```
name — имя пользователя.
state — состояние (present для создания, absent для удаления).
groups — список групп, в которые входит пользователь.
shell — оболочка пользователя (например, /bin/bash, /sbin/nologin).
password — зашифрованный пароль (ссылка на Vault).
ssh_key_url — URL для загрузки публичного SSH-ключа.
expiry_date — дата истечения срока действия в формате YYYY-MM-DD.
password_lock — блокировка пароля (yes/no, опционально).
```

### 5. Запуск плейбука
### Примечания
- Убедитесь, что хост `192.168.1.10` доступен по SSH.

Запустите роль на целевом хосте:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --vault-password-file vault_pass.txt
```
Плейбук поддерживает теги:
```
setup: Установка зависимостей (curl, cron, PowerShell).
groups: Создание групп
users: Управление пользователями (создание, обновление, удаление, отключение).
expiry: Настройка скрипта деактивации и cron.
ssh: Управление SSH-ключами пользователей
```
Пример с указанием тега:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --tags users --vault-password-file vault_pass.txt
```

## Тестирование
Все команды выполняются на чистой Ubuntu Server 22 LTS после настройки SSH-доступа (замените `192.168.1.10` на реальный IP хоста).

### Установка зависимостей
Проверяет установку пакетов `curl`, `cron` и `powershell`:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --tags setup --vault-password-file vault_pass.txt
```
Проверка:
```
ssh ubuntu@192.168.1.10 "dpkg -l | grep -E 'curl|cron|powershell'"
```

### Создание групп
Создает группы `admins` и `developers`:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --tags groups --vault-password-file vault_pass.txt
```
Проверка:
```
ssh ubuntu@192.168.1.10 "getent group | grep -E 'admins|developers'"
```


### Управление пользователями
Создает пользователей `global_admin`, `dev1`, `local_user` с разными группами и перезаписывает права:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --tags users --vault-password-file vault_pass.txt
```
Проверка:
```
ssh ubuntu@192.168.1.10 "getent passwd | grep -E 'global_admin|dev1|local_user'"
ssh ubuntu@192.168.1.10 "id global_admin"
```

### Добавление SSH-ключей
Добавляет SSH-ключ из URL:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --tags ssh --vault-password-file vault_pass.txt
```

Проверка:
```
ssh ubuntu@192.168.1.10 "cat /home/dev1/.ssh/authorized_keys"
```

### Отключение по дате
Настраивает cron и PowerShell-скрипт для отключения/активации пользователей по дате:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --tags expiry --vault-password-file vault_pass.txt
```

Тест:
1. Установите `expiry_date` в прошлом (например, измените в `inventory/hosts.yml` на `"2025-03-27"`).
2. Запустите плейбук снова:
```
ansible-playbook -i inventory/hosts.yml setup_users.yml --tags expiry --vault-password-file vault_pass.txt
```
3. Выполните скрипт вручную:
```
ssh ubuntu@192.168.1.10 "pwsh -File /usr/local/bin/expire_users.ps1"
```
4. Проверьте, что `dev1` отключен:
```
ssh ubuntu@192.168.1.10 "passwd -S dev1"
```

## Структура проекта
- `inventory/hosts.yml` — тестовый инвентарь с тремя уровнями вложенности.
- `setup_users.yml` — основной плейбук.
- `roles/users/` — основная роль.
- `vault.yml` — секреты (зашифрованы).

```
.
├── inventory
│   └── hosts.yml
├── playbooks
│   └── setup_users.yml
├── README.md
├── roles
│   └── users
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       ├── tasks
│       │   └── main.yml
│       └── templates
│           └── expire_users.ps1.j2
├── vault_pass.txt
└── vault.yml
```

