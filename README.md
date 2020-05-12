# Open Register of Interests — deploy

```
docker-compose up -d
docker-compose run --rm worker memorious list
docker-compose run --rm worker memorious run scraper_name
docker-compose exec declaration_nav python /code/oroi/manage.py migrate
docker-compose exec declaration_nav python /code/oroi/manage.py load_scrape_data postgresql://datastore:datastore@datastore/datastore scraper_name
```

Visit http://localhost:8001/ in a browser.
