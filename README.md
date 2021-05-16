# Setting up Immunity with TLS enabled
This document describes how to setup Kong and Immunity enforcing TLS between them using Docker.

## Requirements
- Certificate files for Kong
- Certificate files for Immunity
- CA Root certificate

To follow this guide you need TLS certificate(s) for Kong and Immunity. Here we assume *Kong*
will run at `https://kong.domain.com` and *Immunity* will run at `https://immunity.domain.com`.

If you have a wild card certificate, e.g. `*.domain.com` you can use the same certificate for both
Kong and Immunity by naming them accordingly. For instance, `immunity.domain.com` and `kong.domain.com`

The CA Root certificate is used both by Kong and Immunity to validate certificates received in a TLS
handshake. The location of this file depends on the OS. For example, in Arch Linux it's located at `/etc/ssl/certs/ca-certificates.crt`

## Docker network setup
Let's start by creating a docker network so containers can communicate with each other:
```
docker network create kong
```

## Setting up Kong with TSL
To setup Kong we need to run a Postgres instance and Kong itself.

Starting PostgreSQL:
```
docker run -d \
  --net kong \
  --name kong-database \
  --env POSTGRES_DB=kong \
  --env POSTGRES_USER=kong \
  --env POSTGRES_PASSWORD=kong \
  postgres:12
```

When PostgreSQL is up you can start Kong. Note that except for `KONG_LICENSE_DATA`, all kong environment
variables are defined in `kong.env` file (make sure to update KONG_ADMIN_GUI_URL). Also the following commands assume certificate files are
available at `./kong.domain.com/` and follow Let's Encrypt naming convention.
```
docker run -d \
  --net kong \
  --name kong.domain.com \
  -p 8000:8000/tcp \
  -p 8001:8001/tcp \
  -p 8002:8002/tcp \
  -p 8443:8443/tcp \
  -p 8444:8444/tcp \
  -p 8445:8445/tcp \
  --env-file ./kong.env \
  --env KONG_LICENSE_DATA=${KONG_LICENSE_DATA} \
  -v $PWD/kong.domain.com/fullchain.pem:/etc/kong/certificates/kong.crt \
  -v $PWD/kong.domain.com/privkey.pem:/etc/kong/certificates/kong.key \
  -v /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.pem \
  registry.kongcloud.io/kong-ee-dev-master:latest \
  sh -c "/usr/local/bin/kong migrations bootstrap && /usr/local/bin/kong start"
```

Finally, let's start the upstream that Kong will be routing traffic to:
```
docker run -d --net kong --name store-api kong/testendpoints:latest
```

## Setting up Immunity with TLS
Now we need to run, Redis, another PostgreSQL instance and the Immunity components.

Let's start by running Redis and PostgreSQL:
```
docker run -d --net kong --name redis redis:5.0-alpine

docker run -d \
  --net kong \
  --name collector-database \
  --env POSTGRES_DB=collector \
  --env POSTGRES_USER=collector \
  --env POSTGRES_PASSWORD=collector \
  postgres:12
```


After Redis and PostgreSQL are running we can run *Collector*, Immunity's HTTP server that receives
traffic data from Kong. Same as above, the following command assumes Immunity's certificate files
are available at `./immunity.domain.com` and follow Let's Encrypt naming convention.
```
docker run -d \
  --net kong \
  --name immunity.domain.com \
  -p 5000:5000/tcp \
  --env KONG_PROTOCOL=https \
  --env KONG_HOST=kong.domain.com \
  --env KONG_PORT=8444 \
  --env CELERY_BROKER_URL=redis://redis:6379/0 \
  --env SQLALCHEMY_DATABASE_URI=postgresql://collector:collector@collector-database:5432/collector \
  --env KONG_ADMIN_TOKEN=averygoodtoken \
  --env REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.pem \
  -v $PWD/immunity.domain.com/fullchain1.pem:/etc/collector-cert.crt \
  -v $PWD/immunity.domain.com/privkey1.pem:/etc/collector-key.key \
  -v /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.pem:ro \
  kong/immunity:4.1.0
```

Immunity does its work in tasks dispatched by a scheduler and *collector* server we just executed.
The tasks then are executed by worker. Let's now start the scheduler and the worker:
```
docker run -d \
  --net kong \
  --name immunity-scheduler \
  --env CELERY_BROKER_URL=redis://redis:6379/0 \
  --env SQLALCHEMY_DATABASE_URI=postgresql://collector:collector@collector-database:5432/collector \
  kong/immunity:4.1.0 \
  sh -c "celery beat -l info -A collector.scheduler.celery"


docker run -d \
  --net kong \
  --name immunity-worker \
  --env TRAFFIC_ALERT_MINL=1 \
  --env KONG_PROTOCOL=https \
  --env KONG_HOST=kong.domain.com \
  --env KONG_PORT=8444 \
  --env CELERY_BROKER_URL=redis://redis:6379/0 \
  --env SQLALCHEMY_DATABASE_URI=postgresql://collector:collector@collector-database:5432/collector \
  --env KONG_ADMIN_TOKEN=averygoodtoken \
  --env REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.pem \
  -v /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.pem:ro \
  kong/immunity:4.1.0 \
  sh -c "celery worker -l info -A collector.scheduler.celery --concurrency=1"
```

## Local DNS configuration
In order to call both Kong and Immunity APIs with the proper DNS name, we can configure the host OS
to map the certificate(s) names to localhost. With that, instead of calling kong at
`https://localhost:8444` you can call it at, for instance, `https://kong.domain.com:8444`.
In Linux and MacOS this can be achieved by adding the following lines to `/etc/hosts`:
```
127.0.0.1 immunity.domain.com
127.0.0.1 kong.domain.com
```

## Configuring Kong
From the commands above, we have:
- *Collector* running at `https://immunity.domain.com:5000`
- *Kong* running at `https://kong.domain.com:8443` (proxy), and `https://kong.domain.com:8444` (admin API)

The following commands will create a workspace with a service and a route exposing the upstream API:

```
curl -X POST https://kong.domain.com:8444/workspaces/ --header kong-admin-token:averygoodtoken -d name=sales

curl -X POST https://kong.domain.com:8444/sales/services \
    --header kong-admin-token:averygoodtoken \
    -d name=buy \
    -d url=http://store-api:6000/buy

curl -X POST https://kong.domain.com:8444/sales/services/buy/routes \
    --header kong-admin-token:averygoodtoken \
    -d name=buy-http-route \
    --data-urlencode 'paths=/buy'
```

Now let's enable *Collector* plugin on `sales` workspace:
```
curl -X POST https://kong.domain.com:8444/sales/plugins \
    --header kong-admin-token:averygoodtoken \
    -d name=collector \
    -d config.http_endpoint=https://immunity.domain.com:5000 \
```

## Testing
Let's make a request and check it's data have been sent to Immunity.
```
curl https://kong.domain.com:8443/buy?item_id=210
```

Data about this request should be available with `curl https://immunity.domain.com:5000`
