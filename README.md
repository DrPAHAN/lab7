# Лабораторная работа №7: Серверный редирект по IP-префиксам РФ

## Дисциплина: Компьютерные сети  
## Задание: Реализовать на уровне веб-сервера автоматический редирект клиентов с российских IP-адресов на статическую страницу-заглушку `«ВАМ СЮДА НЕЛЬЗЯ»`. Решение должно работать на уровне сервера, не затрагивая код приложений.
---

## Требования и окружение
| Компонент | Значение / Примечание |
|-----------|-----------------------|
| ОС | macOS 12 Monterey |
| Терминал | `zsh` (по умолчанию в macOS) |
| Пакетный менеджер | Homebrew |
| Веб-сервер | Nginx (установлен через Homebrew) |
| Архитектура | Инструкция написана для Intel (`/usr/local`) |

---

## Установка и настройка

### 1. Установка Nginx
```
brew install nginx
```

### 2. Подготовка списка IP-префиксов РФ
Скачиваем публичный агрегированный список диапазонов и конвертируем его в формат Nginx:
```
mkdir -p /usr/local/etc/nginx/geo /usr/local/var/www

# Скачивание CIDR-списка
curl -sL https://www.ipdeny.com/ipblocks/data/aggregated/ru-aggregated.zone \
  -o /usr/local/etc/nginx/geo/ru_raw.ip

# Преобразование в формат `IP/CIDR 1;` (для Geo)
awk 'NF && !/^#/ {print $1 " 1;"}' /usr/local/etc/nginx/geo/ru_raw.ip \
  > /usr/local/etc/nginx/geo/ru.ip
```

### 3. Создание страницы-заглушки
```
cat > /usr/local/var/www/stub.html << 'EOF'
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>Доступ ограничен</title>
  <style>
    body { font-family: sans-serif; text-align: center; margin-top: 15%; background: #f4f4f4; }
    h1 { color: #d32f2f; }
  </style>
</head>
<body>
  <h1>ВАМ СЮДА НЕЛЬЗЯ</h1>
  <p>Доступ с территории РФ ограничен на уровне сервера.</p>
</body>
</html>
EOF
```

### 4. Конфигурация Nginx
```
nano /usr/local/etc/nginx/nginx.conf
```

Фрагмент добавлен в блок http { ... }
```
    geo $is_ru {
        default 0;
        include /usr/local/etc/nginx/geo/ru.ip;
    }

    server {
        listen       8080;
        server_name  localhost;
        root         /usr/local/var/www;

        location = /stub.html {
            try_files $uri =404;
        }

        location / {
            if ($is_ru) {
                return 302 /stub.html;
            }
            try_files $uri $uri/ =404;
        }
    }
```

### 5. Запуск
Проверка синтаксиса
```
nginx -t
```

Очистка возможных "залипших" процессов
```
brew services stop nginx 2>/dev/null || true
rm -f ~/Library/LaunchAgents/homebrew.mxcl.nginx.plist
sudo pkill nginx 2>/dev/null || true
brew services cleanup
```

Запуск сервиса
```
brew services start nginx
brew services list | grep nginx
```

### 6. Тестирование
```
# 1. Временное добавление localhost в список РФ
echo "127.0.0.1/32 1;" >> /usr/local/etc/nginx/geo/ru.ip
nginx -s reload

# 2. Проверка заголовка редиректа
curl -I http://localhost:8080

# 3. Проверка тела заглушки
curl http://localhost:8080/stub.html

# 4. Удаление тестовой записи и перезагрузка
sed -i '' '/127.0.0.1\/32 1;/d' /usr/local/etc/nginx/geo/ru.ip
nginx -s reload
```

### Принцип работы

| Этап | Описание |
| -----| ---------|
| Приём соединения | Nginx слушает порт 8080 и принимает TCP-запрос |
| Фаза GEO | Директива geo проверяет IP клиента против загруженного CIDR-списка. Данные компилируются в оптимизированное дерево, поиск занимает O(1) |
| Принятие решения | При совпадении $is_ru = 1 директива return 302 генерирует HTTP-ответ до передачи запроса в любой бэкенд |
| Отдача заглушки | Запрос на /stub.html обрабатывается статическим location, минуя if-условия |
| Серверный уровен | Приложение (PHP/Node/Python и т.д.) полностью изолировано от запросов с российских IP. Фильтрация происходит на уровне сетевого стека Nginx |
