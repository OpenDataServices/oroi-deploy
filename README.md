# Open Register of Interests — deploy

## Dev setup

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
