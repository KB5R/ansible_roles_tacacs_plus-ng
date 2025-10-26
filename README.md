# Ansible роль: tac_plus-ng

Универсальная роль для автоматической установки и настройки TACACS+ сервера [tac_plus-ng](http://www.pro-bono-publico.de/projects/) на ALT Linux 9/10.

## Оглавление

- [Что такое TACACS+ и зачем он нужен](#что-такое-tacacs-и-зачем-он-нужен)
- [Что делает эта роль](#что-делает-эта-роль)
- [Требования](#требования)
- [Быстрый старт](#быстрый-старт)
- [Структура роли](#структура-роли)
- [Переменные](#переменные)
- [Методы установки](#методы-установки)
- [Режимы авторизации](#режимы-авторизации)
- [Примеры использования](#примеры-использования)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## Что такое TACACS+ и зачем он нужен

**TACACS+** (Terminal Access Controller Access-Control System Plus) - это протокол для централизованной аутентификации, авторизации и учета (AAA) доступа к сетевому оборудованию.

### Простыми словами:

Представьте, что у вас есть 100 маршрутизаторов и коммутаторов. Без TACACS+ вам нужно:
- Создавать учетные записи на каждом устройстве вручную
- Помнить 100 разных паролей (или использовать один везде - небезопасно)
- При увольнении сотрудника удалять его аккаунт на всех 100 устройствах

С TACACS+ вы:
- Создаете ONE учетную запись на сервере TACACS+
- Все устройства проверяют доступ через этот сервер
- Заблокировали пользователя на сервере = нет доступа ко всем устройствам

### Что умеет tac_plus-ng:

- **Аутентификация**: проверка логина/пароля
- **Авторизация**: определение прав доступа (что можно делать на устройстве)
- **Учет (Accounting)**: логирование всех действий пользователей
- Интеграция с LDAP (FreeIPA, Active Directory)
- Локальная аутентификация (shadow файл)

---

## Что делает эта роль

Эта Ansible роль **автоматизирует** весь процесс установки и настройки tac_plus-ng сервера:

1. ✅ Устанавливает tac_plus-ng (из пакета или исходников)
2. ✅ Настраивает службу systemd
3. ✅ Создает конфигурационные файлы
4. ✅ Настраивает интеграцию с LDAP (FreeIPA или Active Directory)
5. ✅ Настраивает локальных пользователей
6. ✅ Запускает и включает службу

**Без роли:** вам нужно выполнить ~50 команд вручную, настроить конфиги, разобраться в синтаксисе.
**С ролью:** один запуск `ansible-playbook` - и всё готово!

---

## Требования

### Операционная система:
- ALT Linux 9
- ALT Linux 10

### Ansible:
- Ansible >= 2.9
- Python >= 3.6

### Права доступа:
- Root доступ на целевой сервер (или sudo)

### Сетевые требования:
- Доступ к репозиториям ALT Linux (для установки пакетов)
- Доступ к LDAP серверу (если используете FreeIPA/AD авторизацию)

---

## Быстрый старт

### Шаг 1: Установите роль

```bash
# Клонируйте репозиторий
git clone <URL_РЕПОЗИТОРИЯ>
cd tacacs_plus_ng

# Или скопируйте роль в директорию roles вашего проекта
cp -r roles/tac_plus-ng /path/to/your/ansible/project/roles/
```

### Шаг 2: Создайте inventory файл

```ini
# inventory/hosts
[tacacs_servers]
tacacs01.example.com ansible_user=root
```

### Шаг 3: Создайте playbook

```yaml
# playbook.yml
---
- name: Установить и настроить TACACS+ сервер
  hosts: tacacs_servers
  become: true
  roles:
    - tac_plus-ng
```

### Шаг 4: Настройте переменные

```yaml
# inventory/group_vars/tacacs_servers.yml
---
# Метод установки
tacacs_install_method: package  # или 'source'

# Режим авторизации
tacacs_auth_mode: local  # или 'freeipa', 'ad'

# Настройки сети
tacacs_listen_address: 192.168.1.10
tacacs_listen_port: 49
```

### Шаг 5: Запустите playbook

```bash
ansible-playbook -i inventory/hosts playbook.yml
```

**Готово!** TACACS+ сервер установлен и запущен.

---

## Структура роли

```
tac_plus-ng/
├── README.md              # Этот файл - документация
├── defaults/
│   └── main.yml          # Переменные по умолчанию
├── files/                # Статические файлы (НЕ в git - содержит секреты!)
│   ├── tac_plus-ng.service        # Systemd unit для source установки
│   ├── tac_plus_hosts.cfg         # Конфигурация сетевых устройств
│   ├── tac_plus_users.cfg         # Конфигурация пользователей
│   └── shadow.txt                 # Хеши паролей локальных пользователей
├── handlers/
│   └── main.yml          # Обработчики событий (перезапуск сервиса)
├── meta/
│   └── main.yml          # Метаданные роли
├── tasks/
│   ├── main.yml          # Главный файл задач
│   ├── install.yml       # Логика выбора метода установки
│   ├── install_package.yml   # Установка из пакета
│   ├── install_source.yml    # Установка из исходников
│   └── configure.yml     # Настройка конфигурации
└── templates/            # Jinja2 шаблоны конфигураций
    ├── tac_plus-ng_FREEIPA.cfg.j2           # FreeIPA авторизация
    ├── tac_plus-ng_INTEGRATION_AD_MICROSOFT.cfg.j2  # Active Directory
    └── tac_plus-ng_ONLY_LOCAL_USER.cfg.j2  # Локальная авторизация
```

### Как это работает:

1. **Ansible** читает `playbook.yml` и видит роль `tac_plus-ng`
2. Загружается `defaults/main.yml` - переменные по умолчанию
3. Переменные из `inventory/group_vars/` переопределяют defaults
4. Выполняются задачи из `tasks/main.yml`:
   - `install.yml` - проверяет `tacacs_install_method` и вызывает нужный метод
   - `install_package.yml` или `install_source.yml` - устанавливают tac_plus-ng
   - `configure.yml` - создает конфиги из templates
5. При изменении конфигов срабатывает `notify` → запускается handler → перезапуск службы

---

## Переменные

### Основные переменные

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `tacacs_install_method` | `package` | Метод установки: `package` (из репозитория) или `source` (компиляция) |
| `tacacs_auth_mode` | `freeipa` | Режим авторизации: `freeipa`, `ad`, `local` |
| `tacacs_listen_address` | `0.0.0.0` | IP адрес для прослушивания (0.0.0.0 = все интерфейсы) |
| `tacacs_listen_port` | `49` | Порт TACACS+ (стандартный 49) |
| `tacacs_service_enabled` | `true` | Автозапуск службы при загрузке системы |
| `tacacs_service_state` | `started` | Состояние службы: `started`, `stopped` |

### Переменные для пакетной установки

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `tacacs_package_name` | `tac_plus-ng` | Имя пакета в репозитории |
| `tacacs_package_state` | `latest` | Версия пакета: `latest`, `present`, `1.2.3` |
| `tacacs_extra_packages` | `[rsync]` | Дополнительные пакеты для установки |

### Переменные для установки из исходников

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `tacacs_source_archive` | `{{ role_path }}/files/DEVEL.202310061820.tar.bz2` | Путь к архиву с исходниками |
| `tacacs_source_unpack_dir` | `/root` | Куда распаковывать архив |
| `tacacs_build_dependencies` | `[libpcre2-devel, gcc, make, ...]` | Пакеты для компиляции |
| `tacacs_configure_extra_args` | `""` | Дополнительные аргументы для ./configure |

### Переменные LDAP (FreeIPA / Active Directory)

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `tacacs_ldap_base_search_1` | `""` | Адрес первого LDAP сервера (например, `dc01.example.com`) |
| `tacacs_ldap_base_search_2` | `""` | Адрес второго LDAP сервера (резервный) |
| `tacacs_ldap_port` | `389` | Порт LDAP (389 без SSL, 636 с SSL) |
| `tacacs_ldap_bind_user_password` | `""` | Пароль служебного пользователя LDAP |
| `tacacs_ldap_DC_1` | `""` | Домен верхнего уровня (например, `local`) |
| `tacacs_ldap_DC_2` | `""` | Домен второго уровня (например, `company`) |
| `tacacs_ldap_bind_user_ad` | `""` | Логин служебного пользователя AD |
| `tacacs_ldap_bind_user_ipa` | `""` | Логин служебного пользователя FreeIPA |

### Переменные файлов конфигурации

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `tacacs_hosts_src` | `tac_plus_hosts.cfg` | Имя файла с настройками сетевых устройств |
| `tacacs_users_src` | `tac_plus_users.cfg` | Имя файла с настройками пользователей |
| `tacacs_shadow_src` | `shadow.txt` | Имя файла с хешами паролей |
| `tacacs_deploy_shadow` | `true` | Деплоить ли shadow файл |

### Переменные прав доступа

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `tacacs_owner` | `root` | Владелец конфигурационных файлов |
| `tacacs_group` | `root` | Группа конфигурационных файлов |
| `tacacs_config_mode` | `0640` | Права на конфигурационные файлы |
| `tacacs_shadow_mode` | `0600` | Права на shadow файл (только root) |

---

## Методы установки

Роль поддерживает два метода установки tac_plus-ng:

### 1. Установка из пакета (рекомендуется)

**Когда использовать:**
- ✅ Пакет tac_plus-ng доступен в репозитории ALT Linux
- ✅ Вам нужна стабильная версия
- ✅ Вы хотите простоту обновления

**Как настроить:**
```yaml
tacacs_install_method: package
tacacs_package_state: latest  # или конкретная версия
```

**Что происходит:**
1. Роль выполняет `apt-get install tac_plus-ng`
2. Systemd unit уже включен в пакет
3. Роль только настраивает конфигурацию

**Преимущества:**
- Быстрая установка
- Автоматические обновления через apt
- Меньше зависимостей

---

### 2. Установка из исходников

**Когда использовать:**
- ✅ Нужна конкретная версия, которой нет в репозитории
- ✅ Нужны патчи или модификации
- ✅ Пакет в репозитории устарел

**Как настроить:**
```yaml
tacacs_install_method: source
tacacs_source_archive: "{{ role_path }}/files/DEVEL.202310061820.tar.bz2"
tacacs_source_unpack_dir: /root
```

**Что происходит:**
1. Устанавливаются зависимости для компиляции (gcc, make, libpcre2, ...)
2. Распаковывается архив в `tacacs_source_unpack_dir`
3. Выполняется `./configure --prefix=/opt/tac_plus-ng`
4. Патчится Makefile.obj (замена ${LIB_CRYPT} на -lcrypt)
5. Выполняется `make`
6. Выполняется `make install` в `/opt/tac_plus-ng`
7. Деплоится systemd unit из `files/tac_plus-ng.service`

**Преимущества:**
- Полный контроль над версией
- Можно применять патчи
- Установка в кастомную директорию

**Недостатки:**
- Дольше устанавливается
- Нужны пакеты для компиляции
- Обновления только вручную

---

## Режимы авторизации

Роль поддерживает три режима авторизации пользователей:

### 1. Локальная авторизация (local)

**Когда использовать:**
- ✅ Нет LDAP/AD инфраструктуры
- ✅ Небольшое количество пользователей
- ✅ Тестовый стенд

**Как настроить:**
```yaml
tacacs_auth_mode: local
tacacs_deploy_shadow: true
```

**Как работает:**
1. Пароли хранятся в файле `shadow.txt` в виде хешей
2. Пользователи настраиваются в `tac_plus_users.cfg`
3. Аутентификация происходит локально, без LDAP

**Создание пользователя:**
```bash
# Сгенерировать хеш пароля
mkpasswd -m sha-512 "password123"
# Вывод: $6$xyz...abc

# Добавить в files/shadow.txt:
username:$6$xyz...abc:19500:1:99999:0:::
```

**Файл users:**
```
# files/tac_plus_users.cfg
user admin {
    member = admin_group
    password login = mavis
}

user readonly {
    member = ro_group
    password login = mavis
}
```

---

### 2. FreeIPA авторизация

**Когда использовать:**
- ✅ У вас развернут FreeIPA
- ✅ Централизованное управление пользователями
- ✅ Интеграция с существующей инфраструктурой

**Как настроить:**
```yaml
tacacs_auth_mode: freeipa

# LDAP настройки
tacacs_ldap_base_search_1: ipa01.example.com
tacacs_ldap_base_search_2: ipa02.example.com
tacacs_ldap_port: 389
tacacs_ldap_DC_1: com
tacacs_ldap_DC_2: example
tacacs_ldap_bind_user_ipa: tacacs_service
tacacs_ldap_bind_user_password: "YourSecurePassword"
tacacs_ldap_freeipa_base_prefix: cn=users,cn=accounts
```

**Как работает:**
1. TACACS+ подключается к FreeIPA через LDAP
2. При логине ищется пользователь: `uid=username,cn=users,cn=accounts,dc=example,dc=com`
3. Проверяется пароль
4. Определяются группы пользователя
5. На основе групп применяются права доступа

**Требования:**
- Служебный пользователь в FreeIPA с правами чтения
- Доступ к LDAP портам (389/636)
- Правильный base DN

---

### 3. Active Directory авторизация

**Когда использовать:**
- ✅ У вас Windows AD инфраструктура
- ✅ Пользователи уже есть в AD
- ✅ Интеграция с корпоративным каталогом

**Как настроить:**
```yaml
tacacs_auth_mode: ad

# LDAP настройки
tacacs_ldap_base_search_1: dc01.example.com
tacacs_ldap_base_search_2: dc02.example.com
tacacs_ldap_port: 389
tacacs_ldap_DC_1: com
tacacs_ldap_DC_2: example
tacacs_ldap_bind_user_ad: tacacs_svc@example.com
tacacs_ldap_bind_user_password: "YourSecurePassword"
tacacs_ldap_ad_base_prefix: OU=Users,OU=Company
```

**Как работает:**
1. TACACS+ подключается к AD через LDAP
2. При логине ищется пользователь: `(&(objectclass=user)(sAMAccountName=username))`
3. Проверяется пароль
4. Определяются группы AD пользователя
5. На основе групп применяются права доступа

**Требования:**
- Служебный пользователь в AD с правами чтения
- Доступ к контроллерам домена (389/636 порт)
- Правильный base DN и OU

---

## Примеры использования

### Пример 1: Простая установка с локальными пользователями

```yaml
# playbook.yml
---
- name: Установить TACACS+ с локальной авторизацией
  hosts: tacacs_servers
  become: true
  roles:
    - role: tac_plus-ng
      vars:
        tacacs_install_method: package
        tacacs_auth_mode: local
        tacacs_listen_address: 192.168.1.10
```

**Результат:** TACACS+ сервер работает на 192.168.1.10:49, пользователи из shadow файла.

---

### Пример 2: Установка с FreeIPA

```yaml
# inventory/group_vars/tacacs_servers.yml
---
tacacs_install_method: package
tacacs_auth_mode: freeipa
tacacs_listen_address: 10.0.1.50

# FreeIPA настройки
tacacs_ldap_base_search_1: ipa01.company.local
tacacs_ldap_base_search_2: ipa02.company.local
tacacs_ldap_port: 389
tacacs_ldap_DC_1: local
tacacs_ldap_DC_2: company
tacacs_ldap_bind_user_ipa: tacacs_service
tacacs_ldap_bind_user_password: "YourSecurePassword"
```

**Результат:** TACACS+ интегрирован с FreeIPA, пользователи берутся из LDAP.

---

### Пример 3: Установка из исходников с Active Directory

```yaml
# inventory/host_vars/tacacs01.yml
---
tacacs_install_method: source
tacacs_source_archive: "{{ role_path }}/files/DEVEL.202310061820.tar.bz2"
tacacs_auth_mode: ad

# Active Directory настройки
tacacs_ldap_base_search_1: dc01.corp.local
tacacs_ldap_base_search_2: dc02.corp.local
tacacs_ldap_port: 389
tacacs_ldap_DC_1: local
tacacs_ldap_DC_2: corp
tacacs_ldap_bind_user_ad: CN=tacacs_svc,OU=Service Accounts,DC=corp,DC=local
tacacs_ldap_bind_user_password: "YourSecurePassword"
tacacs_ldap_ad_base_prefix: OU=Users,OU=Company
```

**Результат:** TACACS+ скомпилирован из исходников, интегрирован с AD.

---

### Пример 4: Несколько TACACS+ серверов (HA)

```yaml
# inventory/hosts
[tacacs_servers]
tacacs01.example.com tacacs_listen_address=10.0.1.51
tacacs02.example.com tacacs_listen_address=10.0.1.52

# playbook.yml
---
- name: Развернуть отказоустойчивый кластер TACACS+
  hosts: tacacs_servers
  become: true
  roles:
    - role: tac_plus-ng
      vars:
        tacacs_install_method: package
        tacacs_auth_mode: freeipa
        tacacs_ldap_base_search_1: ipa01.example.com
        tacacs_ldap_base_search_2: ipa02.example.com
```

**Результат:** Два TACACS+ сервера с одинаковой конфигурацией, высокая доступность.

---

## Troubleshooting

### Проблема: Служба не запускается

**Симптом:**
```
TASK [tac_plus-ng : Управление службой tac_plus-ng] ****
fatal: [tacacs01]: FAILED! => {"msg": "Service tac_plus-ng not found"}
```

**Причина:** Systemd unit не создан.

**Решение:**
```bash
# Проверьте наличие unit файла
ssh root@tacacs01 "ls -la /etc/systemd/system/tac_plus-ng.service"

# Если файла нет - проверьте переменную
tacacs_manage_service: true

# И метод установки (для source нужно деплоить unit вручную)
tacacs_install_method: source
```

---

### Проблема: LDAP подключение не работает

**Симптом:**
```
# В логах /var/log/tac_plus-ng/authc/...
LDAP: Connection failed
```

**Диагностика:**
```bash
# Проверьте доступность LDAP сервера
telnet dc01.example.com 389

# Проверьте учетные данные
ldapsearch -H ldap://dc01.example.com:389 \
  -D "CN=tacacs_svc,OU=Service Accounts,DC=example,DC=com" \
  -w "password" \
  -b "DC=example,DC=com" \
  "(sAMAccountName=testuser)"
```

**Решение:**
- Проверьте `tacacs_ldap_bind_user_password`
- Проверьте `tacacs_ldap_DC_1` и `tacacs_ldap_DC_2`
- Проверьте firewall правила (порт 389/636)

---

### Проблема: Каталоги для логов не существуют

**Симптом:**
```
# В логах systemd
Failed to open log file: No such file or directory
```

**Причина:** Директории `/var/log/tac_plus-ng/*` не созданы.

**Решение:**
```bash
# Создайте вручную
ssh root@tacacs01 "mkdir -p /var/log/tac_plus-ng/{authz,authc,acct}"

# Или добавьте в роль задачу (уже есть в рекомендациях)
```

---

### Проблема: Ошибка при компиляции из исходников

**Симптом:**
```
TASK [tac_plus-ng : Собрать tac_plus-ng] ****
fatal: [tacacs01]: FAILED! => {"msg": "make: *** [tac_plus-ng] Error 1"}
```

**Причина:** Не хватает зависимостей для компиляции.

**Решение:**
```bash
# Проверьте установлены ли зависимости
ansible tacacs_servers -m package -a "name=gcc,make,libpcre2-devel state=present"

# Проверьте логи компиляции
ssh root@tacacs01 "cat /root/PROJECTS/tac_plus-ng/config.log"
```

---

### Проблема: Permission denied при доступе к shadow файлу

**Симптом:**
```
# В логах
mavis_tacplus_shadow.pl: Permission denied: /opt/tac_plus-ng/etc/shadow
```

**Причина:** Неправильные права на shadow файл.

**Решение:**
```bash
# Установите правильные права
ssh root@tacacs01 "chmod 600 /opt/tac_plus-ng/etc/shadow"
ssh root@tacacs01 "chown root:root /opt/tac_plus-ng/etc/shadow"
```

---

## FAQ

### Q: Можно ли использовать роль для обновления существующего сервера?

**A:** Да! Роль идемпотентна - можно запускать многократно:
- Если tac_plus-ng уже установлен - переустановки не будет (если `package_state: present`)
- Конфигурационные файлы обновятся
- Служба перезапустится только при изменении конфигов

---

### Q: Как изменить порт TACACS+?

**A:** Установите переменную:
```yaml
tacacs_listen_port: 2049  # Вместо стандартного 49
```

---

### Q: Можно ли использовать и LDAP, и локальных пользователей одновременно?

**A:** Да! Роль поддерживает fallback:
1. Сначала проверяется LDAP
2. Если пользователь не найден в LDAP - проверяется shadow файл

Оставьте `tacacs_deploy_shadow: true` и добавьте локальных пользователей в shadow.

---

### Q: Как добавить нового пользователя без перезапуска роли?

**A:**
```bash
# Для локальных пользователей:
# 1. Добавить в files/tac_plus_users.cfg
# 2. Обновить files/shadow.txt с хешем пароля
# 3. Скопировать на сервер
ansible tacacs_servers -m copy -a "src=files/tac_plus_users.cfg dest=/opt/tac_plus-ng/etc/"
ansible tacacs_servers -m systemd -a "name=tac_plus-ng state=reloaded"

# Для LDAP - просто создайте пользователя в FreeIPA/AD
```

---

### Q: Как настроить SSL/TLS для LDAP?

**A:** Измените порт на 636:
```yaml
tacacs_ldap_port: 636
```

И используйте `ldaps://` в `tacacs_ldap_base_search_1`:
```yaml
tacacs_ldap_base_search_1: ldaps://dc01.example.com
```

---

### Q: Роль поддерживает другие дистрибутивы кроме ALT Linux?

**A:** Нет, роль написана специально для ALT Linux 9/10. Для других дистрибутивов потребуются изменения:
- Имена пакетов
- Пути к конфигурационным файлам
- Команды установки

---

### Q: Как проверить что TACACS+ сервер работает?

**A:**
```bash
# Проверить статус службы
ansible tacacs_servers -m systemd -a "name=tac_plus-ng"

# Проверить слушает ли порт
ansible tacacs_servers -m shell -a "netstat -tulpn | grep 49"

# Проверить логи
ansible tacacs_servers -m shell -a "tail -20 /var/log/tac_plus-ng/authc/$(date +%Y/%m/%d).log"
```

---

### Q: Как сделать backup конфигурации?

**A:**
```bash
# Backup конфигов
ansible tacacs_servers -m archive -a "path=/opt/tac_plus-ng/etc dest=/root/tacacs_backup_$(date +%Y%m%d).tar.gz"

# Скачать на control node
ansible tacacs_servers -m fetch -a "src=/root/tacacs_backup_*.tar.gz dest=./backups/"
```

---

## Автор

**Mihail Popkov**

## Лицензия

GPL

## Поддержка

При возникновении проблем:
1. Проверьте раздел [Troubleshooting](#troubleshooting)
2. Проверьте логи: `/var/log/tac_plus-ng/`
3. Создайте issue в репозитории с подробным описанием проблемы

---

## Дополнительные ресурсы

- [Официальная документация tac_plus-ng](http://www.pro-bono-publico.de/projects/)
- [RFC 8907: TACACS+ Protocol](https://tools.ietf.org/html/rfc8907)
- [Ansible Documentation](https://docs.ansible.com/)

