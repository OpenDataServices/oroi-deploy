# oroi-proxy (Reverse Proxy for declared.info)

## Live deploy (docker stack)

First run the [docker-workarounds salt state](https://github.com/OpenDataServices/opendataservices-deploy/blob/master/salt/docker-workarounds.sls).

``` bash
docker swarm init
git clone https://github.com/OpenDataServices/oroi-deploy
docker stack deploy -c oroi-deploy/oroi-proxy/docker-compose.yml oroi-proxy
```

