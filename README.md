# Proxmox Backup Telegram Notifier

Система мониторинга бэкапов Proxmox VE с уведомлениями в Telegram на русском языке.

## Что это такое

**Автоматический мониторинг** всех задач резервного копирования (vzdump) в Proxmox VE с отправкой **детальных отчетов в Telegram**. Скрипт работает как системная служба, анализирует логи бэкапов и переводит все ошибки на понятный русский язык.

## Как это работает

```
Proxmox API → Анализ задач vzdump → Определение статуса VM → Telegram уведомления
     ↓              ↓                        ↓                     ↓
Каждые 15с    Извлечение логов    Перевод ошибок на русский    Отчеты и алерты
```

**Процесс:**
1. **Мониторинг API** - каждые 15 секунд проверяет новые/завершенные задания vzdump
2. **Анализ логов** - извлекает статус каждой VM/CT из детальных логов  
3. **Умное распознавание** - определяет тип ошибки из 50+ паттернов
4. **Уведомления** - отправляет итоговые отчеты и индивидуальные ошибки

## Основной функционал

### 📊 **Итоговые отчеты**
- Сводка по завершению каждого задания бэкапа
- Статус всех VM/CT с размерами архивов и временем выполнения
- Информация о производительности и использовании хранилища
- Предупреждения при заполненности хранилищ >90%

### 🚨 **Мониторинг ошибок** 
- Индивидуальные уведомления для каждой проблемной VM
- **50+ типов ошибок** с переводом на русский:
  - QMP timeout ошибки → "тайм-аут QMP команды backup"
  - Fleece image ошибки → "ошибка флисинг образа (уже существует)"  
  - Storage problems → "хранилище nfs-backup не существует"
  - Disk space issues → "нехватка места на диске при записи"
  - Container errors → "ошибка контейнера (tar permission denied)"
  - VM состояния → "VM приостановлена — не удалось остановить"

### 🔍 **Мониторинг в реальном времени**
- Отслеживание running заданий с ошибками
- Уведомления об остановке заданий пользователем  
- Поддержка прерываний (Ctrl+C, SIGTERM)

### 🛠️ **Диагностика и логирование**
- DEBUG режим для детальной диагностики проблем
- Автоматическая ротация и очистка логов
- Отслеживание обработанных заданий (без дубликатов)

## Преимущества

### ✅ **Полнота и точность**
- **Мониторинг всех аспектов** бэкапа - не пропускает ни одной VM
- **Интеллектуальный анализ** - правильно определяет причины сбоев
- **Без ложных срабатываний** - умная фильтрация и проверка недавности

### ✅ **Удобство использования**  
- **Русскоязычные уведомления** - все ошибки переведены понятным языком
- **Структурированные отчеты** - четкая информация по каждой VM
- **Поддержка топиков** - организация уведомлений в Telegram группах

### ✅ **Надежность**
- **Автоматическое восстановление** после сбоев API или сети
- **Минимальная нагрузка** - оптимизированные запросы к API  
- **Безопасность** - минимальные права доступа, защита токенов

### ✅ **Гибкость настройки**
- **Кластер или узел** - мониторинг всего кластера или отдельного сервера
- **Настраиваемые пороги** - временные рамки и уровни предупреждений
- **DEBUG режим** - подробная диагностика для решения проблем

## Возможности

- **Мониторинг бэкапов** - отслеживание задач vzdump в реальном времени
- **Детальные отчеты** - статус каждой VM/CT с размерами и временем
- **50+ типов ошибок** - QMP timeout, fleece errors, storage issues, disk space
- **Умный анализ** - автоматический перевод ошибок на русский
- **Предупреждения хранилища** - контроль заполненности
- **DEBUG режим** - детальное логирование для диагностики

## Установка

### 1. Подготовка системы
```bash
apt update && apt install curl jq -y
```

### 2. Telegram бот
1. Создайте бота через @BotFather
2. Получите Chat ID через @getmyid_bot
3. Для групповых чатов - добавьте бота в группу

### 3. API токен Proxmox
```
Datacenter → Permissions → API Tokens → Add
User: root@pam
Token ID: backup-monitor  
Privilege Separation: отключить
```

### 4. Установка скрипта
```bash
# Создание директории
mkdir -p /opt/proxmox-notifier

# Размещение скрипта
nano /opt/proxmox-notifier/proxmox-telegram-notifier.sh
# Вставьте содержимое скрипта

# Права доступа
chmod +x /opt/proxmox-notifier/proxmox-telegram-notifier.sh
```

### 5. Настройка скрипта
```bash
nano /opt/proxmox-notifier/proxmox-telegram-notifier.sh
```

**Измените параметры:**
```bash
PROXMOX_API_URL="https://YOUR-IP:8006/api2/json"
PROXMOX_API_TOKEN="PVEAPIToken=root@pam!backup-monitor=YOUR-SECRET"
PROXMOX_NODE=""  # Пусто для кластера, или имя узла
TELEGRAM_BOT_TOKEN="YOUR-BOT-TOKEN"
TELEGRAM_CHAT_ID="YOUR-CHAT-ID"
TELEGRAM_THREAD_ID="2"  # Опционально для топиков
```

### 6. Создание systemd службы

**Создайте файл службы:**
```bash
nano /etc/systemd/system/proxmox-telegram-notifier.service
```

**Содержимое службы:**
```ini
[Unit]
Description=Proxmox Backup Telegram Notifier v3.9
Documentation=man:systemd.service(5)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/opt/proxmox-notifier/proxmox-telegram-notifier.sh
Restart=always
RestartSec=30
StandardOutput=journal
StandardError=journal

# Директории
WorkingDirectory=/opt/proxmox-notifier
RuntimeDirectory=proxmox-notifier
RuntimeDirectoryMode=0755

# Безопасность
NoNewPrivileges=yes
ProtectHome=yes
ProtectSystem=strict
ReadWritePaths=/var/log /tmp /run/proxmox-notifier

# Ресурсы
MemoryMax=256M
TasksMax=50

[Install]
WantedBy=multi-user.target
```

### 7. Запуск службы
```bash
# Перезагрузка конфигурации
systemctl daemon-reload

# Включение автозапуска
systemctl enable proxmox-telegram-notifier.service

# Запуск службы
systemctl start proxmox-telegram-notifier.service

# Проверка статуса
systemctl status proxmox-telegram-notifier.service
```

## Управление службой

### Основные команды
```bash
# Запуск
systemctl start proxmox-telegram-notifier.service

# Остановка
systemctl stop proxmox-telegram-notifier.service

# Перезапуск
systemctl restart proxmox-telegram-notifier.service

# Статус (детальный)
systemctl status proxmox-telegram-notifier.service

# Включить автозапуск
systemctl enable proxmox-telegram-notifier.service

# Отключить автозапуск
systemctl disable proxmox-telegram-notifier.service
```

### Просмотр логов
```bash
# Логи службы в реальном времени
journalctl -u proxmox-telegram-notifier.service -f

# Логи приложения в реальном времени  
tail -f /var/log/proxmox-telegram-notifier.log

# Последние 50 строк
journalctl -u proxmox-telegram-notifier.service -n 50

# Логи за сегодня
journalctl -u proxmox-telegram-notifier.service --since today

# Логи с ошибками
journalctl -u proxmox-telegram-notifier.service -p err
```

## Файлы и директории

### Структура установки
```
/opt/proxmox-notifier/
├── proxmox-telegram-notifier.sh    # Основной скрипт
└── README.md                       # Документация (опционально)

/etc/systemd/system/
└── proxmox-telegram-notifier.service   # Файл службы

/var/log/
├── proxmox-telegram-notifier.log       # Основной лог
└── proxmox-telegram-notifier.log.*     # Архивы логов

/tmp/
├── proxmox_backup_sent_tasks.txt       # Отправленные задания
├── proxmox_backup_sent_errors.txt      # Отправленные ошибки
└── proxmox_notifier_last_cleanup       # Маркер очистки
```

### Логирование
- **Системные логи:** `journalctl -u proxmox-telegram-notifier.service`
- **Лог приложения:** `/var/log/proxmox-telegram-notifier.log`
- **DEBUG логи:** включаются через `DEBUG_ENABLED=true`
- **Ротация:** автоматическая при 50 МБ

## Запуск и тестирование

### Тестовый запуск
```bash
# Запуск в консоли (для отладки)
/opt/proxmox-notifier/proxmox-telegram-notifier.sh

# Включить DEBUG для диагностики
sed -i 's/DEBUG_ENABLED=false/DEBUG_ENABLED=true/' /opt/proxmox-notifier/proxmox-telegram-notifier.sh
```

### Проверка работы
```bash
# 1. Статус службы
systemctl is-active proxmox-telegram-notifier.service

# 2. Последние логи
journalctl -u proxmox-telegram-notifier.service --since "5 minutes ago"

# 3. Процесс работает
pgrep -f proxmox-telegram-notifier.sh

# 4. Проверка подключения к API
curl -k -H "Authorization: PVEAPIToken=root@pam!backup-monitor=SECRET" \
  "https://YOUR-IP:8006/api2/json/version"

# 5. Тест Telegram
curl -X POST "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<CHAT_ID>&text=Тест"
```

## Примеры уведомлений

### Успешный бэкап
```
✅ Завершение задания бэкапа 📊

🔧 ПАРАМЕТРЫ ЗАПУСКА
      • Кластер: production
      • Узел: pve-node1
      • Завершен: 10.09.2025 03:45:15

📊 РЕЗУЛЬТАТЫ
      • Всего объектов: 3
      • Успешно: 3
      • С ошибками: 0

🎯 Успешные ВМ/CT
      • VM 101 (web-server) — 00:25:14, 45.2GB, [Стоп]
      • VM 102 (db-server) — 00:18:42, 28.9GB, [Снапшот]
      • CT 201 (nginx) — 00:03:18, 2.8GB, [Стоп]
```

### Бэкап с ошибками  
```
⚠️ Завершение задания бэкапа 📊

📊 РЕЗУЛЬТАТЫ
      • Всего объектов: 3
      • Успешно: 1
      • С ошибками: 2

🎯 Успешные ВМ/CT
      • VM 101 (web-server) — 00:12:15, 15.3GB, [Стоп]

⚠️ Неуспешные ВМ/CT
      • VM 102 (db-server) — тайм-аут QMP команды backup
      • VM 103 (file-server) — ошибка флисинг образа (уже существует)

⚠️ ПРЕДУПРЕЖДЕНИЕ О ХРАНИЛИЩЕ
      • Хранилище backup-storage заполнено на 89%
      • Рекомендуется очистка старых бэкапов
```

### Индивидуальная ошибка
```
🚨 Ошибка бэкапа VM/LXC

🖥️ Виртуальная машина
      • ID: 106
      • Имя: critical-db
      • Тип: VM

❌ Детали ошибки
      • Причина: нехватка места на диске при записи
      • Время начала: 10.09.2025 02:00:15
      • Время ошибки: 10.09.2025 02:18:42
```

## Настройки скрипта

### Основные параметры
```bash
# Интервалы
CHECK_INTERVAL=15                    # Проверка каждые 15 секунд
RECENT_TASK_THRESHOLD=3600           # Обрабатывать задачи не старше 1 часа
MAX_TASK_AGE=7200                    # Макс. возраст running задач (2 часа)

# Функции
DEBUG_ENABLED=true                   # Детальные логи
CLEANUP_ENABLED=true                 # Автоочистка старых файлов
STORAGE_WARNING_ENABLED=true         # Предупреждения о хранилище
STORAGE_WARNING_THRESHOLD=90         # Порог предупреждения (90%)
```

## Устранение проблем

### Служба не запускается
```bash
# Проверить синтаксис
bash -n /opt/proxmox-notifier/proxmox-telegram-notifier.sh

# Проверить права
ls -la /opt/proxmox-notifier/proxmox-telegram-notifier.sh

# Проверить статус
systemctl status proxmox-telegram-notifier.service -l
```

### Нет уведомлений  
```bash
# Включить DEBUG
sed -i 's/DEBUG_ENABLED=false/DEBUG_ENABLED=true/' /opt/proxmox-notifier/proxmox-telegram-notifier.sh

# Перезапустить
systemctl restart proxmox-telegram-notifier.service

# Смотреть DEBUG логи
tail -f /var/log/proxmox-telegram-notifier.log | grep DEBUG
```

### Ошибки API/Telegram
```bash
# Тест API Proxmox
curl -k -H "Authorization: $TOKEN" "$API_URL/cluster/tasks"

# Тест Telegram
curl "https://api.telegram.org/bot$BOT_TOKEN/getMe"
```

## Обновление

### Обновление скрипта
```bash
# Остановить службу
systemctl stop proxmox-telegram-notifier.service

# Заменить скрипт новой версией
nano /opt/proxmox-notifier/proxmox-telegram-notifier.sh

# Запустить службу
systemctl start proxmox-telegram-notifier.service
```

### Очистка логов
```bash
# Очистить старые логи (опционально)
rm -f /var/log/proxmox-telegram-notifier.log.*

# Очистить рабочие файлы
rm -f /tmp/proxmox_backup_sent_*
```
