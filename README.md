## Хакатон PlaysDev

### 2 таска

Репозиторий содержит config-и для настроенного Nginx сервера

----------------------------------------

На главной странице лежат сссылки на   
* информацию про PHP сервер (`host/info.php`)
* перехода на другую .hmtl страничку, что обрабатывается этим же nginx
* переход на другой сайт
* редирект на `/redblue` 

`redblue` - это обратный прокси с балансировкой нагрузки по типу Round Robbin. Проксируются серверы, поднятые на 8081 и 8082 портах `localhost`

Так же можно воспользоваться следующими запросами   

| Запрос  | Что происходит |
| --------|  ---------------|
| /localhost/music | скачивает музыку из каталога `www/data` |
| /localhost/info.php | проксирует запрос на 81 порт с PHP |
| /localhost/secondserver | проксирует запрос на `localhost` 5500 порт |
| /localhost/*.jpg | переворачивает картинку и выводит. картинка должна лежать в `var/www/nginx/image` |
| /localhost\*.png | выводит картинку. картинка должна лежать в `var/www/nginx/image` |

------------------------------------


Во время работы Nginx логи записываются в `/var/log/nginx/access.log`   

Разработан **daemon-процесс**, который будет раз в несколько секунд писать логи в файл `log1`. При достижении `log1` размера больше 300Кб он очищается, и в файл `log2` записывается информация о времени очистки и количестве удаленных записей

Все ошибочные логи записываются в `/var/log/nginx/error.log`. Daemon-процесс все логи с кодом 4хх записывает в файл `log4`, а с кодом 5xx - `log3`.

Настройка произведена следующим образом:

файл `/usr/local/bin/log_rotate.sh`
```bash
#!/bin/bash

# Пути к логам
access_log="/var/log/nginx/access.log"
error_log="/var/log/nginx/error.log"
log1="/var/www/nginx/log1.log"
log2="/var/www/nginx/log2.log"
log3="/var/www/nginx/log3.log"
log4="/var/www/nginx/log4.log"
max_size=300000  # Максимальный размер файла 1 (300 КБ)

# Функция для записи в лог очистки
log_cleanup() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - File cleaned. Removed $1 entries." >> $log2
}

# Очистка лог-файла 1, если он превышает размер
check_and_cleanup_log1() {
    log_size=$(stat --format=%s "$log1")
    if [ "$log_size" -gt "$max_size" ]; then
        removed_lines=$(wc -l < "$log1")
        > "$log1"  # Очищаем файл
        log_cleanup "$removed_lines"
    fi
}

# Парсинг логов и распределение их по файлам
parse_logs() {
    tail -n 1000 "$access_log" | while read -r line; do
        # Добавление в лог 5xx
        if [[ "$line" =~ " 5[0-9][0-9] " ]]; then
            echo "$line" >> "$log3"
        # Добавление в лог 4xx
        elif [[ "$line" =~ " 4[0-9][0-9] " ]]; then
            echo "$line" >> "$log4"
        fi
        # Добавление в лог 1
        echo "$line" >> "$log1"
    done
}

parse_logs2() {
    tail -n 1000 "$error_log" | while read -r line; do
        # Добавление в лог 5xx
        if [[ "$line" =~ " 5[0-9][0-9] " ]]; then
            echo "$line" >> "$log3"
        # Добавление в лог 4xx
        elif [[ "$line" =~ " 4[0-9][0-9] " ]]; then
            echo "$line" >> "$log4"
        fi
        # Добавление в лог 1
        echo "$line" >> "$log1"
    done
}

# Основной цикл демона
while true; do
    parse_logs
    parse_logs2
    check_and_cleanup_log1
    sleep 5  # Задержка 5 секунд
done
```

файл `/etc/systemd/system/log_rotate.service`
```bash
[Unit]
Description=NGINX log rotation daemon
After=nginx.service

[Service]
ExecStart=/usr/local/bin/log_rotate.sh
Restart=always
User=nginx
Group=nginx
RestartSec=3

[Install]
WantedBy=multi-user.target
```
После следует:   
`log_rotate.sh` сделать исполняемым ->
```chmod +x /usr/local/bin/log_rotate.sh```

перезагрузить `systemd` ->
```systemctl daemon-reload```

активировать и запустить службу логирования
```
systemctl enable log_rotate.service
systemctl start log_rotate.service
```



