# oroi-proxy (Reverse Proxy for declared.info)

## Live deploy (docker stack)

``` bash
# First run the docker-workarounds salt state https://github.com/OpenDataServices/opendataservices-deploy/pull/100
docker swarm init
git clone https://github.com/OpenDataServices/oroi-deploy
docker stack deploy -c oroi-deploy/oroi-proxy/docker-compose.yml oroi-proxy
```

