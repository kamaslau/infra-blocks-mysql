# infra-blocks-mysql

[![Blocks](https://img.shields.io/badge/infra--blocks-index-555?style=flat-square)](https://github.com/kamaslau/infra-blocks)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

A unit, not a demo: one service, one port, one documented credential, disposable with a single command.

Typical use: moving a legacy system that was already MySQL, and nobody wants to convert it before it runs.

[MySQL](https://dev.mysql.com/doc/) / [MariaDB](https://mariadb.org/documentation/) with [Adminer](https://www.adminer.org/). For PostgreSQL, see [infra-blocks-postgresql](https://github.com/kamaslau/infra-blocks-postgresql).

## Service URL

- Database (MySQL/MariaDB) `localhost:3306`
- Web UI (Adminer) [http://localhost:8080](http://localhost:8080)

## Default user

Defaults are for local use only.

| Username | Password |
| -------- | -------- |
| root     | 123456   |

## Usage

### Start with [Docker Compose](https://docs.docker.com/compose/)

```bash
# Initiate .env file
cp .env.sample .env
# Start services
docker compose up -d
```

Update existing composed containers with latest images:

```bash
docker compose pull && \
docker compose down && \
docker compose up -d
```

### Start services individually

Launch database first, then the UI.

#### MariaDB

```bash
docker run -p 3306:3306 -d --restart always --env MYSQL_ROOT_PASSWORD=123456 --name mysql mariadb:latest
```

#### MySQL

```bash
docker run -p 3306:3306 -d --restart always --env MYSQL_ROOT_PASSWORD=123456 --name mysql mysql:latest
```

#### Adminer

Pass in to link the database container with param `--link database-container-name:db` .

```bash
docker run -p 8080:8080 -d --restart always --env ADMINER_DEFAULT_SERVER=mysql --name adminer adminer:latest
```

### Further operations

```bash
# Enter container and initiate shell
docker exec -it mysql bash

# List installed plugins
ls -l /usr/lib/mysql/plugin
```

### Error handling

## Host '172.18.0.x' is not allowed to connect to this MySQL server

[用户无权通过当前网络访问 MySQL](https://github.com/docker-library/mysql/issues/275)。

```bash
# Enter container and initiate shell
docker exec -it mysql bash

# Login as root user
mysql -u root -p
```

```sql
-- Create root user visiting from any network with password 123456 (change this for safety)
CREATE USER 'root'@'%' IDENTIFIED BY '123456';
grant all privileges on *.* to 'root'@'%';
-- grant all privileges on *.* to 'root'@'%' identified by '123456';

-- Verify the user creation
select host ,user from mysql.user;
```

## Verify

```bash
docker compose exec -T mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "select version();"'
```

A table with one row comes back, starting `8.4` or `11.x` depending on the tag. Expect a `[Warning] Using a password on the command line interface can be insecure` line on stderr — that is the client, not a failure.

## Why this shape

Adminer is preconfigured with `ADMINER_DEFAULT_SERVER=mysql` so the login form has no server field to get wrong. That single variable is the difference between "open the URL and log in" and a phone call.

The `'root'@'%'` grant stays in the README because the error it fixes — *Host '172.18.0.x' is not allowed to connect* — appears the first time anyone reaches this database from another container or from the LAN, and the message points at networking while the actual cause is a host-scoped account. It costs four lines to keep and has already been looked up more than once.

No charset or collation is pinned. MySQL 8 and MariaDB 11 both default to `utf8mb4`, and pinning a collation by hand is how legacy dumps end up restored in the wrong order. If the source system is 5.7, pin the tag instead of the charset.

## Notes for deployments in China

`mysql:latest`, `mariadb:latest` and `adminer` all need a registry mirror on most mainland hosts. A legacy dump often arrives with `utf8mb3` tables; check `select @@character_set_database` before importing rather than after.

---

Part of [infrastructure blocks](https://github.com/kamaslau/infra-blocks) · ← back to index
