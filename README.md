# Инструкция: Flask + systemd + Nginx (lesson 15)

## Предварительно

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip nginx
sudo useradd -r -s /bin/false webadmin 2>/dev/null || true
```

## 1. Flask-приложение

```bash
sudo mkdir -p /opt/app15
sudo nano /opt/app15/app15.py
```

Вставить код из методички, в конце:

```python
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8015, debug=False)
```

Виртуальное окружение и зависимости:

```bash
sudo python3 -m venv /opt/app15/venv
sudo /opt/app15/venv/bin/pip install --upgrade pip
sudo /opt/app15/venv/bin/pip install flask
sudo chown -R webadmin:webadmin /opt/app15
```

## 2. systemd-сервис

```bash
sudo nano /etc/systemd/system/app15.service
```

```ini
[Unit]
Description=Flask app15 on port 8015
After=network.target

[Service]
Type=simple
User=webadmin
Group=webadmin
WorkingDirectory=/opt/app15
ExecStart=/opt/app15/venv/bin/python /opt/app15/app15.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Запуск:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now app15
sudo systemctl status app15 --no-pager
curl http://127.0.0.1:8015/api/data
```

## 3. Статика

```bash
sudo mkdir -p /var/www/static
cd /var/www/static

sudo nano index.html   # код из задания
sudo nano style.js     # код из задания
sudo wget -O nginx-logo.png https://nginx.org/nginx.png

sudo chown -R www-data:www-data /var/www/static
sudo chmod -R 755 /var/www/static
```

## 4. Nginx как reverse-proxy

```bash
sudo nano /etc/nginx/sites-available/app15
```

```nginx
server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/app15_access.log;
    error_log  /var/log/nginx/app15_error.log;

    location /static/ {
        alias /var/www/static/;
        try_files $uri =404;
        expires 1h;
        add_header Cache-Control "public";
    }

    location / {
        proxy_pass         http://127.0.0.1:8015;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

Активация:

```bash
sudo ln -sf /etc/nginx/sites-available/app15 /etc/nginx/sites-enabled/app15
sudo rm -f /etc/nginx/sites-enabled/default
sudo rm -f /etc/nginx/sites-enabled/flaskapp   # если был битый симлинк
sudo nginx -t
sudo systemctl reload nginx
```

## 5. Проверка

```bash
# Flask напрямую
curl -I http://127.0.0.1:8015/

# Через Nginx
curl -I http://localhost/
curl -I http://localhost/static/index.html
curl -I http://localhost/static/nginx-logo.png
curl -s http://localhost/api/data
time curl -s http://localhost/slow
```

В браузере (с хоста, без порта):

```
http://192.168.2.105/                     <- главная Flask через Nginx
http://192.168.2.105/static/index.html    <- статика (карточки, тёмная тема)
http://192.168.2.105/api/data             <- JSON
http://192.168.2.105/slow                 <- 3 сек
```

## 6. Что важно

| Правило | Почему |
|---|---|
| В браузере **без `:8015`** | порт 8015 — напрямую Flask, он не знает `/static/` -> 404 |
| `proxy_pass` = порт Flask (**8015**) | иначе 502 Bad Gateway |
| `alias /var/www/static/;` **со слэшем** | без слэша путь склеится неправильно -> 404 |
| `debug=False` в `app.run` | иначе автоперезагрузчик конфликтует с systemd |
| `chown -R www-data` на `/var/www/static`, `webadmin` на `/opt/app15` | Nginx читает статику, сервис — код |
| `nginx -t` **и** `systemctl reload nginx` | `-t` только проверяет, не применяет |

## 7. Если что-то не работает

```bash
sudo systemctl status app15 --no-pager
sudo journalctl -u app15 -n 30 --no-pager
sudo tail -n 30 /var/log/nginx/app15_error.log
sudo ss -ltnp | grep -E ':(80|8015) '
sudo nginx -T | grep -A 8 'location /static/'
```

| Симптом | Причина |
|---|---|
| **502** на `/` | Flask не запущен / не на 8015 -> проверь `ss`, `journalctl` |
| **404** на `/static/*` (от Werkzeug) | в браузере `:8015`, убери порт |
| **404** на `/static/*` (от Nginx) | файла нет в `/var/www/static` или `alias` без слэша |
| **403** на `/static/*` | права: `chown -R www-data:www-data /var/www/static` |
| `nginx -t`: `sites-enabled/flaskapp` не найден | битый симлинк: `sudo rm -f /etc/nginx/sites-enabled/flaskapp` |
| `/style.css` 404 в консоли | файла нет в задании — нормально; `sudo touch /var/www/static/style.css` для чистоты |

## 8. Итоговая схема

```
Браузер :80 -> Nginx -┬- /static/* -> /var/www/static/ (сам)
                      └- /*        -> 127.0.0.1:8015 (Flask app15)
```

Всё готово, если: `http://192.168.2.105/` показывает «Welcome to the test app!» с логотипом nginx, а `http://192.168.2.105/static/index.html` — страницу с карточками и рабочей кнопкой тёмной темы.
