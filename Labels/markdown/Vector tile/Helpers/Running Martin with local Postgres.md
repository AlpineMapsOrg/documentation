# Running Martin with local Postgres
```shell
docker run \
  --net=host \
  -e DATABASE_URL=postgresql://postgres:alpinemaps@localhost/alpinemaps \
  ghcr.io/maplibre/martin

# DATABASE_URL SYNTAX:   postgresql://USER:PASSWORD@localhost/DATABASENAME 
```




