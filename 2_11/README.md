# Exercise 2.11: Dev environment

For this exercise I used a project from the KodeKloud e-commerce app (`kodekloudhub/learning-app-ecommerce`). It's a PHP web app that talks to a MariaDB database.

## The problem

To run this app locally without containers, I'd have to install Apache, PHP 7.4 and MariaDB on my laptop, create the database, load the SQL script by hand, and hope my versions match what the app expects. A new person joining the project would have to repeat all of that.

## What I did

The goal is the same idea as in the course: someone who clones the repo only needs Docker and Git.

**1. A Dockerfile for the web app** (`./app`)

The web image is built on `php:7.4-apache`, so the PHP and Apache versions are pinned in the image instead of depending on whatever is installed on the host. I also added a `.dockerignore` to keep files that don't belong in the image out of the build context.

**2. A `docker-compose.yml` with two services**

```yaml
services:
  mysql-service:
    image: mariadb:10.11
    ...
  web:
    build: ./app
    ...
volumes:
  db-data:
```

- `mysql-service`: the official `mariadb:10.11` image.
- `web`: built from `./app`. Inside the Compose network it can reach the database by the service name `mysql-service`.

### The mounts

The database service has two mounts, and they do different jobs.

**Bind mount for the seed script**

```yaml
volumes:
  - type: bind
    source: ./db/db-load-script.sql
    target: /docker-entrypoint-initdb.d/01-db-load-script.sql
    read_only: true
    bind:
      create_host_path: false
```

The MariaDB image runs every `.sql` file it finds in `/docker-entrypoint-initdb.d/` when the database is created for the first time. By mounting `db-load-script.sql` there, the shop's tables and data are loaded automatically on first start, with no manual steps. Some details:

- It's a **bind mount** because the file is part of the project and lives in the repo. If I edit it on my machine, the container sees the change.
- `read_only: true` means the container can only read the file and can't change it on my host.
- `create_host_path: false` makes Compose fail with an error if `./db/db-load-script.sql` doesn't exist. Without it, Docker would silently create an empty *directory* with that name and the database would start with no data.
- The `01-` prefix is there because the scripts in that folder run in alphabetical order, so I can add more scripts before or after it later.
- I used the long syntax (`type`, `source`, `target`) instead of `./db/...:/...` because it's more explicit and it's the only way to set `create_host_path`.

**Named volume for the data**

```yaml
- db-data:/var/lib/mysql
```

This is where MariaDB stores its data files. It's a Docker managed volume (declared under `volumes:` at the bottom), so the data survives `docker compose down` and container rebuilds. Here I don't need to see the files on my host, so a named volume is enough.

The two mounts work together, with one catch: the seed script only runs when `/var/lib/mysql` is empty, so on the first start. After that the data comes from `db-data` and the script is skipped. To start from scratch and load the script again:

```bash
docker compose down -v
docker compose up
```

### Other details

- **Healthcheck.** The database uses `healthcheck.sh --connect --innodb_initialized`, a script that ships with the MariaDB image, checked every 5 seconds with up to 20 retries. The `web` service has `depends_on` with `condition: service_healthy`, so it only starts once the database is really ready, not just started. Plain `depends_on` only controls the start order.
- **No hardcoded credentials.** Passwords and names come from variables like `${DB_PASSWORD}` and `${DB_NAME}`, which Compose reads from a `.env` file or the environment. The compose file can be committed without secrets in it.
- **Port.** The web port is `127.0.0.1::80`. There is no host port after the first colon, so Docker picks a free one, because I don't know which ports are free on every laptop. The `127.0.0.1` part publishes it only on localhost, so the app isn't reachable from other machines on the network.
- **Image name.** `image: ${WEB_IMAGE:-ecom-web:local}` gives the built image a name. It defaults to `ecom-web:local`, and I can override it with `WEB_IMAGE`, for example when tagging it for Docker Hub.

## How to run it

```bash
git clone <repo-url>
cd docker_challenge
# create a .env file with DB_ROOT_PASSWORD, DB_NAME, DB_USER and DB_PASSWORD
docker compose up --build
```

Useful commands while developing:

```bash
docker compose ps              # see if the database is healthy
docker compose logs -f web     # follow the web logs
docker compose logs mysql-service
docker compose port web 80     # find which host port Docker picked
docker compose down            # stop and remove the containers (data is kept)
docker compose down -v         # also remove the data volume
```

The web code is copied into the image at build time (`build: ./app`), not bind mounted, so after changing the PHP code I rebuild with `docker compose up --build`.
