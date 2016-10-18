### Кыргызстан в векторе: cвой сервер векторных карт на базе OSM2VectorTiles

Чтобы настроить свой сервер кекторных карт на базе инструментов OSM2VectorTiles нам понадобиться установленный [Docker](https://docs.docker.com/engine/installation/) и [Docker-compose](https://docs.docker.com/compose/install/).

Скачаем проект и перейдем в его директорию.
```
git clone https://github.com/osm2vectortiles/osm2vectortiles.git
cd ./osm2vectortiles
```

Запустим контейнер с PostGIS.
```
docker-compose up -d postgis
```

Скачаем последние данные Кыргызстана в директорию `import`.
```
wget http://download.geofabrik.de/asia/kyrgyzstan-latest.osm.pbf -P ./import
```

Импортируем внешние ресурсы такие как, полигоны воды взятые с [OpenStreetMapData.com](http://openstreetmapdata.com/data/water-polygons) и [Natural Earth](http://www.naturalearthdata.com) данные для низкого масштаба, метки страны и областей.
```
docker-compose up import-external
```

Импортируем PBF файл Кыргызстана в PostGIS, у меня данная процедура выполнилась минут за 5-8.
```
docker-compose up import-osm
```

Импортируем дополнительный функционал, SQL утилиты необходимые для создания векторных плиток.
```
docker-compose up import-sql
```

Теперь экспортируем MBTiles передав координаты для ограничивающего параллелепипеда, максимального и минимального масштаба.
```
docker-compose run -e BBOX="69.265,39.1728,80.2296,43.2668" -e MIN_ZOOM="0"  -e MAX_ZOOM="22" export
```

В завершении сгенерируем векторные плитки и создадим MBTile файл в директории `export`.
```
docker-compose up export
```

Чтобы увидеть результат, установим и запустим сервер рендера векторных плиток.
```
npm install -g tileserver-gl
tileserver-gl-light kyrgyzstan.mbtiles
```
Или при помощи Docker, команду выполняем в директориии с MBTile файлом.
```
docker run -it -v $(pwd):/data -p 8080:80 klokantech/tileserver-gl
```

Результат можно наблюдать по адресу `http://localhost:8080`, по умолчанию нам будет доступна карта в двух стилях.

![Maps](http://i.imgur.com/dgQljSy.png)
