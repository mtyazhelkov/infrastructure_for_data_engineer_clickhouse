# Инфраструктура для Data-Engineer ClickHouse

Доработан прооект по изначальноей статье на [habr](https://habr.com/ru/articles/842818/)


Запуск docker-compose.yaml: docker-compose up -d
Остановка docker-compose.yaml: sudo docker-compose stop
Вывод логов: docker-compose logs


Архитектура папок

/infrastructure_for_data_engineer_clickhouse/
docker-compose.yaml      # Главный файл запуска
.env                     #  Файл, где лежит CLICKHOUSE_PASSWORD
make_backup.sh           # Скрипт автоматизации бэкапов
ch_data/                 # Данные таблиц (монтируется в /var/lib/clickhouse)
ch_backups/              # Сами файлы бэкапов .zip (хост-папка)
ch_config/               # Конфигурация (монтируется в config.d)
config.xml           # Сеть, порты и лимиты RAM
backup_disk.xml      # Описание диска 'backups'
allow_backup_disk.xml # Разрешение на использование этого диска

В backup_disk.xml мы указали путь /var/lib/clickhouse/backups/. 
Благодаря  docker-compose.yaml, эта папка внутри контейнера зеркально связана с папкой ./ch_backups на  хосте.

Последовательность команд:

1 создание YAML-файла docker_compose.yaml


2 Создание структуры папок на хосте
  # Папка для данных БД
  mkdir -p ./ch_data

  # Папка для конфигов (согласно вашему yaml)
  mkdir -p ./ch_config

  # Папка для самих бэкапов
  mkdir -p ./ch_backups

3 Создание файлов конфигурации
Теперь создаем те самые файлы внутри ./ch_config:
# Файл 1: Описание диска для бэкапов
 ./ch_config/backup_disk.xml

# Файл 2: Разрешение на использование диска
 ./ch_config/allow_backup_disk.xml

# Файл 3: Сеть и лимиты памяти
 ./ch_config/config.xml

4 Настройка прав и перезапуск
ClickHouse внутри контейнера должен иметь права на запись в папку на хосте.

# Выдаем права на папки

# Выдаем права владельца (101 - это clickhouse)
sudo chown -R 101:101 ./ch_data ./ch_backups ./ch_config
sudo chmod -R 755 ./ch_config


# Запуск (используем up -d, чтобы поднять сервис)
docker-compose up -d

5 Скрипт автоматизации (make_backup.sh)

#!/bin/bash
# Путь к папке, где лежит docker-compose.yml
BASE_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$BASE_DIR"

# Имя файла бэкапа
FILENAME="backup_$(date +%Y%m%d_%H%M%S).zip"

CLICKHOUSE_PASSWORD="пароль"

# 1. Запуск бэкапа внутри контейнера
docker exec clickhouse clickhouse-client \
  --user click \
  --password "${CLICKHOUSE_PASSWORD}" \
  --query "BACKUP ALL TO Disk('backups', '$FILENAME')"

# 2. Удаление старых бэкапов (старше 7 дней) в папке хоста
find ./ch_backups -name "backup_*.zip" -mtime +7 -delete

Делаем скрипт исполняемым chmod +x make_backup.sh

6 Ручной запуск бэкапа (проверка)


Ручная команда для создания бэкап архива:
docker exec -it clickhouse clickhouse-client \
  --user click \
  --password "${CLICKHOUSE_PASSWORD}" \
  --query "BACKUP DATABASE default TO Disk('backups', 'backup_$(date +%F_%H-%M).zip')"

Файл появится в вашей локальной папке ./ch_backups.


7 Автоматизация (Cron-скрипт ставим на расписание)

Делаем скрипт исполняемым chmod +x /infrastructure_for_data_engineer_clickhouse/make_backup.sh

Добавление в расписание
Откройте планировщик crontab -e и добавьте строку для ежедневного запуска в 03:00 ночи
0 3 * * * /bin/bash /infrastructure_for_data_engineer_clickhouse/make_backup.sh >> /var/log/ch_backup.log 2>&1

8 Финальная проверка связи:

# Проверка TCP порта 9000
docker exec -it clickhouse clickhouse-client --user click --password ПАРОЛЬ --query "SELECT version(), 1"

# Проверка HTTP порта 8123
curl http://localhost:8123/

