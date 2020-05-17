### Настройка балансировки FreeIPA серверов.
#### Задачи:
```
1) Балансировка трафика на 80/443/tcp порт для доступа к доступному GUI интерфейсу.
2) Балансировка трафика на порты 636/tcp/udp (LDAPS), 389/tcp/tcp/udp (LDAP), 88/tcp/udp (Kerberos), 464/tcp/udp (Kerberos), 123/tcp/udp (NTP) 
```

#### Реализация: 
1)
 -------------------------------------------------------------------------------

``` ini
#### Т.к nginx умеет проксировать udp трафик то реализуем на нем.



# 1.1 Задаем upstream группу.
# В моей реализации все три ноды в режиме multimaster!

upstream ipa_servers {
    ip_hash; # Привязываем сессию, что бы при каждом запросе не улетало на новый бек.
    
    server ipa.internal-server-01:443 max_fails=3 fail_timeout=10s; 
    server ipa.internal-server-02:443 max_fails=3 fail_timeout=10s;  
    server ipa.internal-server-03:443 max_fails=3 fail_timeout=10s;
}

#!!!! Фронтенд не сможет сам переключиться т.к JS не понимает, что нода умерла и нужно переключиться,
# поэтому скорее всего вы увидите долгий loader. Решение: полностью обновить страницу.

# 1.2 Добавляем редирект с 80/tcp на 443/tcp

server {
    listen 80;
    server_name ipa.external-example.com;
    return 301 https://$host$request_uri;
}

# 1.3 Добавляем блок HTTPS

server {
    listen 443 ssl;
    server_name ipa.external-example.com;

    access_log /var/log/nginx/ipa-access.log ipa;
    error_log /var/log/nginx/ipa-error.log ;

    ssl_certificate /etc/letsencrypt/live/<ipa.external-example.com>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<ipa.external-example.com>/privkey.pem; 

location / {
    rewrite ^/$ https://$host/ipa/ui/; 
    proxy_pass              https://ipa_servers/; # proxy to upstream group
    proxy_set_header        Host $host;
    proxy_set_header        Referer ipa; # Можно установить любой, главное заменить этот реферер на стороне freeipa-apache.
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto https;
    proxy_connect_timeout   150;
    proxy_send_timeout      100;
    proxy_read_timeout      100;
    proxy_buffers           4 32k;
    client_max_body_size    200M;
    client_body_buffer_size 512k;
    keepalive_timeout       5;
    add_header              Strict-Transport-Security max-age=63072000;
    add_header              X-Frame-Options DENY;
    add_header              X-Content-Type-Options nosniff;
 }
}

# 1.4 Правим конфиг на стороне Freeipa.

## Открываем файл /etc/httpd/conf.d/nss.conf и добавляем две строчки в блок <VirtualHost _default_:443>
ProxyPassReverseCookieDomain ipa.internal-server-01 ipa.external-example.com;
RequestHeader edit Referer ^ipa https://ipa.internal-server-01/ipa/ui/

# Для каждого бека свой internal server.
```
2)
-------------------------------------------------------------------
```ini
# Открываем /etc/nginx/nginx.conf и после блока http добавляем блок stream {}
# Для каждого порта прописываем аналогичную конструкцию.

stream {
    upstream dns_upstreams {
        least_conn;
        server ip_1:389;
        server ip_2:389;
	    server ip_3:389;
    }

    server {
        listen 389 udp  reuseport;
	    listen 389; #tcp
        proxy_pass dns_upstreams;
        proxy_timeout 5s;
        proxy_responses 2;
        error_log /var/log/nginx/<service>.log;
    }

```

# Если есть ошибки или замечания готов выслушать и если, что поправить конфигурацию.