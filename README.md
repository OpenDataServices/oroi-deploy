# Open Register of Interests — deploy

Note, there are two sets of commands here, docker-compose commands for running a local dev copy, and docker stack commands for running a live copy on a server.

## Dev setup (docker-compose)

### Getting started

``` bash
# Run most containers (all except those that initiate/do a scrape/load)
docker-compose -f docker-compose.yml -f docker-compose.override-dev.yml up -d
# Trigger memorious scrapes
docker-compose -f docker-compose-memorious-run.yml up -d
# Follow these with
docker-compose logs -f memorious-worker
# Once they're done (or sooner if you want to test a partial load)
docker-compose -f docker-compose-declaration_nav-load.yml up -d
# You can follow the logs at:
docker-compose -f docker-compose-declaration_nav-load.yml logs -f declaration_nav-load
# And follow the logs of the web ui at:
docker-compose logs -f declaration_nav
```

You should be able to visit http://localhost:8001/ in a browser (may take a couple of minutes).

### Pull the latest images

``` bash
docker-compose pull
docker-compose -f docker-compose.yml -f docker-compose.override-dev.yml up -d
```

### Build an image locally

e.g.

``` bash
cd path/to/open-register-of-interests
docker build -t opendataservices/open-register-of-interests:docker .
cd back/to/this/dir
# Reload the container
docker-compose -f docker-compose.yml -f docker-compose.override-dev.yml up -d
```

You should be able to visit http://localhost:8001/ in a browser (may take a couple of minutes).

### Removing everything, so that you can start again

``` bash
# Stop and remove all the containers
docker-compose -f docker-compose.yml -f docker-compose.override-dev.yml -f docker-compose-memorious-run.yml -f docker-compose-declaration_nav-load.yml rm --stop
# Remove any volumes that were associated with the containers
docker volume prune
# Check they're all gone
docker volume ls
```

### Remove all django data, and run the load from scratch

The advantage of these steps is they don't stop the memorious scrape, or delete its db.

``` bash
# Stop and remove the containers
docker-compose -f docker-compose.yml -f docker-compose.override-dev.yml -f docker-compose-declaration_nav-load.yml rm --stop declaration_nav-load declaration_nav django-postgres elasticsearch
# Remove any volumes that were associated with the containers
docker volume prune
# Make sure most containers are running again
docker-compose -f docker-compose.yml -f docker-compose.override-dev.yml up -d
# Run the load
docker-compose -f docker-compose-declaration_nav-load.yml up -d
```

## Live deploy (docker stack)

### Getting started, and run the scrapes

If on Bytemark, set up a server with "Docker on Debian 9" and 5GiB of RAM.
Then, run the docker-workarounds salt state https://github.com/OpenDataServices/opendataservices-deploy/pull/100

``` bash
docker swarm init
git clone https://github.com/OpenDataServices/oroi-deploy
# Run most containers (all except those that initiate/do a scrape/load):
# Rerun this command whenever you have updates to docker images, or the docker
# compose file
docker stack deploy -c oroi-deploy/docker-compose.yml oroi-sprint3
# Trigger memorious scrapes
docker stack deploy -c oroi-deploy/docker-compose-memorious-run.yml oroi-sprint3
# Watch the scrapes:
docker service logs oroi-sprint3_memorious-worker --raw -f --tail 10
```

### Load the data into Django

Once the scrapers are done (or sooner if you want to test a partial load):

``` bash
docker stack deploy -c oroi-deploy/docker-compose-declaration_nav-load.yml oroi-sprint3
# Watch the load:
docker service logs oroi-sprint3_declaration_nav-load --raw -f --tail 10
# Check for errors with:
docker service logs oroi-sprint3_declaration_nav-load | grep Error
# Rebuild ES index:
docker exec --tty -i $(docker ps -q -f name=oroi-sprint3_declaration_nav.1) ./manage.py search_index --rebuild -f
# Make CSV:
docker exec --tty -i $(docker ps -q -f name=oroi-sprint3_declaration_nav.1) sh -c './manage.py csv_user_dump_all && mv /tmp/all_data.csv /django-static/static'
```

### Pull the latest images

This will happen automatically when you run a `docker stack deploy` command.

### Re-doing bits

Redeploy with new images or changes to the compose file:
``` bash
docker stack deploy -c oroi-deploy/docker-compose.yml oroi-sprint3
```

Remove all persistent data relating to scraping (all memorious data, but not that loaded into Django), and relaunch the scrapers:
``` bash
docker service rm oroi-sprint3_memorious-postgres oroi-sprint3_memorious-redis oroi-sprint3_memorious-worker oroi-sprint3_memorious-run
docker volume rm oroi-sprint3_memorious-data oroi-sprint3_memorious-postgres oroi-sprint3_memorious-redis
docker stack deploy -c oroi-deploy/docker-compose.yml -c oroi-deploy/docker-compose-memorious-run.yml oroi-sprint3
```

Remove all django data:
``` bash
docker service rm oroi-sprint3_declaration_nav oroi-sprint3_declaration_nav-load oroi-sprint3_django-postgres oroi-sprint3_elasticsearch tmp-postgres oroi-sprint3_apache-static
# Wait a minute for these to shut down properly
docker volume rm oroi-sprint3_django-postgres oroi-sprint3_elasticsearch
docker stack deploy -c oroi-deploy/docker-compose.yml oroi-sprint3
```
Then, to reload follow [Load the data into Django](#load-the-data-into-django) above.

### Postgres database import/export

Before any of these:

``` bash
# Make the directory we'll mount inside the docker container
mkdir /mnt/docker-export
# Remove the service name we want to use, in case it already exists
docker service rm tmp-postgres
```

After any of these (to watch the progress):
``` bash
# See if the service is still running:
docker service ps tmp-postgres
# Watch the logs (won't exit once done):
docker service logs --raw -f tmp-postgres
```

Postgres export — memorious db:
``` bash
docker service create --name tmp-postgres --restart-condition=none --detach --network=oroi-sprint3_default --mount type=bind,source=/mnt/docker-export,destination=/export postgres:11.4 sh -c 'pg_dump "host=memorious-postgres user=datastore password=datastore" > /export/db.log && ls -lh /export/db.log'
```

Postgres export — django db:
``` bash
docker service create --name tmp-postgres --restart-condition=none --detach --network=oroi-sprint3_default --mount type=bind,source=/mnt/docker-export,destination=/export postgres:11.4 sh -c 'pg_dump "host=django-postgres user=django_db password=django_db" > /export/db.log && ls -lh /export/db.log'
```

Postgres import — memorious db:
``` bash
docker service create --name tmp-postgres --restart-condition=none --detach --network=oroi-sprint3_default --mount type=bind,source=/mnt/docker-export,destination=/export postgres:11.4 psql "host=memorious-postgres user=datastore password=datastore" -f /export/oroi-scrape-sprint3-attempt06.sql
```

Postgres import — django db:
``` bash
docker service create --name tmp-postgres --restart-condition=none --detach --network=oroi-sprint3_default  --mount type=bind,source=/mnt/docker-export,destination=/export  postgres:11.4 psql "host=django-postgres user=django_db password=django_db" -f /export/oroi-django-sprint3-attempt01.sql
```

### Interactive postgres SQL session

`docker stack` doesn't support interactive sessions, so we do a raw `docker exec` with a bash command substitution to work out the container id.

Memorious db:
``` bash
docker exec --tty -i $(docker ps -q -f name=oroi-sprint3_memorious-postgres.1) psql -h 127.0.0.1 -U datastore
```

Django db:
``` bash
docker exec --tty -i $(docker ps -q -f name=oroi-sprint3_django-postgres.1) psql -h 127.0.0.1 -U django_db
```
