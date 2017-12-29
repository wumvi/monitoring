
## Install Config Delivery

Хранилище конфигов
```
docker volume create configs
```

Сохраняем ключ
```
docker run -ti \
    --rm \
    --name config-delivery \
    -v configs:/configs/ \
    -v $(pwd)/keys:/root/.ssh/ \
    wumvi/config-delivery \
    /init-key.sh
```

Получаем конфиги

```
docker run -ti \
    --rm \
    --name config-delivery \
    -v configs:/configs/ \
    -v $(pwd)/keys:/root/.ssh/ \
    wumvi/config-delivery git@domain/branch
```

## Install Grafana
Хранит сертификаты
```
docker volume create domain.ssl
```
Хранит данные мониторинга
```
docker network create monitoring
```

Запускаем образ для автоматического обнавления сертификатов и перезагрузки графаны

```
docker run --restart always \ 
     -ti \
     -d \
     -e CONTAINER=monitoring \
     -v domain.ssl:/etc/letsencrypt/ \
     -v /var/run/docker.sock:/var/run/docker.sock \ 
     -e EMAIL=info@email.com \ 
     --hostname certbot.monintoring \
     --name certbot.monitoring \
     --net monitoring \
     -p 443:443 \
     wumvi/certbot
```

Создаём новый сертификат

```
MONITORIN_DOMAIN=monitoring.domain.com
EMAIL=info@email.com
docker exec \
    -ti \
    certbot.monitoring \
    certbot certonly \
    -agree-tos \
    --email $EMAIL \
    -d $MONITORIN_DOMAIN \
    --standalone 
    --preferred-challenges tls-sni
```

Проверка созданного сертификата

```
docker exec -ti certbot.monitoring ls -la /etc/letsencrypt/live/$MONITORIN_DOMAIN/
```

Меняем права на директорию, чтобы grafana смогла их прочитать

```
docker exec -ti certbot.monitoring chmod -R a+x /etc/letsencrypt/*
```

Запускаем образ grafana

```
GRAFANA_USER=Root
GRAFANA_PWD=`openssl rand -hex 12`
echo Grafana password is $GRAFANA_PWD

docker run -ti --rm \
--name=grafana-xxl \
--hostname grafana-xxl \
-p 3000:3000 \
-v domain.ssl:/etc/letsencrypt/ \
-e GF_DEFAULT_INSTANCE_NAME=$MONITORIN_DOMAIN \
-e GF_SECURITY_ADMIN_USER=$GRAFANA_USER \
-e GF_SECURITY_ADMIN_PASSWORD=$GRAFANA_PWD \
-e GF_SERVER_PROTOCOL=https \
-e GF_SERVER_CERT_FILE=/etc/letsencrypt/live/$MONITORIN_DOMAIN/fullchain.pem \
-e GF_SERVER_CERT_KEY=/etc/letsencrypt/live/$MONITORIN_DOMAIN/privkey.pem \
monitoringartist/grafana-xxl:latest
```

## Install Prometheus

```
docker run --rm \
    -v /root/configs/:/prometheus-data \
    --name=prometheus \
    --hostname prometheus \
    --net=monitoring \
    prom/prometheus --config.file=/prometheus-data/prometheus.yml
    
# -p 9090:9090
```

## Install Exporters

Создаём сеть обсущую сеть
```
docker network create prod.net
```

PostgreSql Exporter
```
POSTGRESQL_USER=postgresql
POSTGRESQL_HOST=host
POSTGRESQL_PWD=pwd

docker run --rm \
    -ti \
    -e DATA_SOURCE_NAME="$POSTGRESQL_USER://postgres:$POSTGRESQL_PWD@$POSTGRESQL_HOST:5432/?sslmode=disable" \
    --net=prod.net \
    --hostname pgexporter \
    --name pgexporter \
    wrouesnel/postgres_exporter
    
# -p 9187:9187
```

Docker Exporter

```
docker run -v /sys/fs/cgroup:/cgroup \
    -v /var/run/docker.sock:/var/run/docker.sock \
    --net prod.net \
    --hostname dockerexporter \
    --name dockerexporter \
    prom/container-exporter
    
# -p 9105:9104
```

Nginx Exporter

```
DOMAIN_VTS=domain
docker run -ti \
    --rm \
    --env NGINX_STATUS="http://$DOMAIN_VTS:8181/status/format/json" \
    --net=prod.net \
    --hostname nginxexporter \
    --name nginxexporter \
    sophos/nginx-vts-exporter
    
# -p 9106:9913 
```

Запуск Sql Агента, нужен для Sql Exporter

```
docker run -ti \
    --rm \
    --name sql-agent.mon \
    --net=prod.net \
    --hostname sqlagent \
    --name sqlagent \
    dbhi/sql-agent
# -p 5000:5000
```

Тест для проверки работы sql-agent

```
curl -H "Content-Type: application/json" -X POST -d '{"driver": "postgres", "connection": {"host": "host", "user": "user", "password":"pwd", "database":"db-name"}, "sql": "SELECT * from users"}' http://IP:5000/
```

Exporter для Sql запросов

```
docker run -ti \
    --rm \
    -v /root/configs/promsql/prometheus-sql.yml:/prometheus-sql.yml \
    -v /root/configs/promsql/queries.yml:/queries.yml \
    --net=prod.net \
    --hostname sqlexporter \
    --name sqlexporter \
    dbhi/prometheus-sql -port 9104 -service http://sqlagent:5000 -config prometheus-sql.yml

# -p 9970:9104
```

Exporter Для Ноды

```
docker run -ti \
    --rm \
    --net=prod.net \
    --hostname nodeexporter \
    --name nodeexporter \
    prom/node-exporter
    
# -p 9100:9100
```

## Install Exporter Gate

Создаём Hash для Basic Authentication

```
HASH=`(PASSWORD="mypassword";SALT="$(openssl rand -base64 3)";SHA1=$(printf "$PASSWORD$SALT" | openssl dgst -binary -sha1 | sed 's#$#'"$SALT"'#' | base64);printf "{SSHA}$SHA1\n")`
```

Сохраняем в файл
```
echo 'mon':$HASH > /etc/.nginx-htpasswd
```

Устанавливаем Exporter Gate
```
docker run -ti \
    --rm \
    --net=prod.net \
    -e DOCKER=on \
    -e NODE=on \
    -e DB=on \
    -e NGINX=on \
    -e SQL=on \
    -p 9130:9130 \
    -v /etc/.nginx-htpasswd:/etc/.nginx-htpasswd
    --hostname gateexporter \
    --name gateexporter \
    wumvi/gate-exporter
```
