# Инфраструктура для Data-Engineer ClickHouse

Статья на [habr](https://habr.com/ru/articles/842818/)

Для того чтобы создать виртуальное окружение для работы с ClickHouse выполните команду:

```bash
python3.12 -m venv venv && \
source venv/bin/activate && \
pip install --upgrade pip && \
pip install -r requirements.txt
```
___

Запуск docker-compose.yaml: docker-compose up -d
Остановка docker-compose.yaml: sudo docker-compose stop
Вывод логов: docker-compose logs
