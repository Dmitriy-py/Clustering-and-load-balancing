# Домашнее задание к занятию 2 «Кластеризация и балансировка нагрузки»

## ` Климов Дмитрий `

## Задание 1

1. Запустите два simple python сервера на своей виртуальной машине на разных портах
2. Установите и настройте HAProxy, воспользуйтесь материалами к лекции по ссылке
3. Настройте балансировку Round-robin на 4 уровне.
На проверку направьте конфигурационный файл haproxy, скриншоты, где видно перенаправление запросов на разные серверы при обращении к HAProxy.


## Ответ:

 ` haproxy.cfg `

```

global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend main
    bind *:80  # Слушаем на порту 80
    default_backend servers

backend servers
    balance roundrobin
    server server1 127.0.0.1:8001 check
    server server2 127.0.0.1:8002 check


```

` server.py `

```
from http.server import HTTPServer, BaseHTTPRequestHandler
import sys

class RequestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        message = "Hello from Python server running on port {}".format(self.server.server_port)
        self.wfile.write(bytes(message, "utf8"))

if __name__ == '__main__':
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 8000
    server_address = ('', port)
    httpd = HTTPServer(server_address, RequestHandler)
    print('Starting server on port {}'.format(port))
    httpd.serve_forever()


```

![Снимок экрана (1056)](https://github.com/user-attachments/assets/65da4fc9-d115-4bdc-8500-40f9704b934d)

![Снимок экрана (1054)](https://github.com/user-attachments/assets/56a2f0b1-6931-4b57-9e41-ed6ebe2df234)

![Снимок экрана (1055)](https://github.com/user-attachments/assets/2969f4f1-e306-49d5-8996-4216a1b8732e)


## Задание 2

1. Запустите три simple python сервера на своей виртуальной машине на разных портах
2. Настройте балансировку Weighted Round Robin на 7 уровне, чтобы первый сервер имел вес 2, второй - 3, а третий - 4
3. HAproxy должен балансировать только тот http-трафик, который адресован домену example.local
На проверку направьте конфигурационный файл haproxy, скриншоты, где видно перенаправление запросов на разные серверы при обращении к HAProxy c использованием домена example.local и без него.

## Ответ:

 ` haproxy.cfg `

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend main
    bind *:80
    # ACL для example.local
    acl is_example_local hdr(host) -i example.local

    # Перенаправляем трафик для example.local в backend servers_example_local
    use_backend servers_example_local if is_example_local

    # Для всего остального трафика отправляем на default_backend (можно настроить другой backend или просто вернуть ошибку)
    default_backend default_backend

backend servers_example_local
    balance roundrobin
    server server1 127.0.0.1:8001 weight 2 check
    server server2 127.0.0.1:8002 weight 3 check
    server server3 127.0.0.1:8003 weight 4 check

backend default_backend
   # В данном случае, возвращаем ошибку.  Можно настроить другой сервер для обработки трафика не example.local
   http-request deny

```

![Снимок экрана (1058)](https://github.com/user-attachments/assets/7347fe9b-c4dd-4210-bbd8-f8b6cb8cf569)

![Снимок экрана (1059)](https://github.com/user-attachments/assets/0c513a51-abf4-4c41-bd97-d98336c8a8d2)


## Задание 3*


1. Настройте связку HAProxy + Nginx как было показано на лекции.
2. Настройте Nginx так, чтобы файлы .jpg выдавались самим Nginx (предварительно разместите несколько тестовых картинок в директории /var/www/), а остальные запросы переадресовывались на HAProxy, который в свою очередь переадресовывал их на два Simple Python server.

На проверку направьте конфигурационные файлы nginx, HAProxy, скриншоты с запросами jpg картинок и других файлов на Simple Python Server, демонстрирующие корректную настройку.

## Ответ:

 ` haproxy.cfg `

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend main
    bind *:81  # Слушаем на порту 81 (порт, на который Nginx перенаправляет запросы)
    default_backend servers

backend servers
    balance roundrobin
    server server1 127.0.0.1:8001 check
    server server2 127.0.0.1:8002 check

```

` /etc/nginx/sites-available/default `

```
server {
    listen 80;
    server_name 192.168.0.117; # Или IP адрес вашей машины

    root /var/www/example.com/public_html;
    index index.html index.htm index.nginx-debian.html;

    location ~* \.jpg$ {
        try_files $uri $uri/ =404;
    }

    location / {
        proxy_pass http://127.0.0.1:81;  # Перенаправляем запросы на HAProxy (порт 81)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

![Снимок экрана (1061)](https://github.com/user-attachments/assets/7bfeadcf-bb96-4ed5-9f2e-3bfe05ce0db6)

![Снимок экрана (1062)](https://github.com/user-attachments/assets/4f7b14c5-3924-4a00-b551-a6ae377b1950)

![Снимок экрана (1063)](https://github.com/user-attachments/assets/e6253e92-85b8-42c8-9c10-a9919c5a2bab)
