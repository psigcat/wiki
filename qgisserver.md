# Instalación y configuración QGIS Server

1. [Instalar QGIS](#instalar-qgis)
2. [Configurar Nginx](#configurar-nginx)
3. [Configurar fastcgi](#configurar-fastcgi)
4. [Configurar postgresql](#configurar-postgresql)
5. [Configurar proyecto QGIS](#configurar-proyecto-qgis)
6. [Ejemplo uso OpenLayers](#ejemplo-uso-openlayers)

## Instalar QGIS

Para instalar un versión actual de QGIS en un servidor Debian/Ubuntu primero hay que añadir el repositorio y la llave tal como esta indicado [aquí](https://qgis.org/en/site/forusers/alldownloads.html#debian-ubuntu)

Después instalamos QGIS Server con: 

`sudo apt-get install qgis-server`

## Configurar Nginx

1. Activar configuración por defecto haciendo enlace simbólico: 

`sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default`

2. Configurar puerto 6080 como default para Nginx cambiando el fichero `/etc/nginx/sites-available/default`:
```
server {
	listen 6080 default_server;
	listen [::]:6080 default_server ipv6only=on;
```

3. Reiniciar Nginx: `sudo service nginx restart`

4. Comprobar acceso: http://localhost:6080/

## Configurar fastcgi

1. Instalar fastcgi: `sudo apt-get install fcgiwrap`
2. Añadir configuración insertando al fichero `/etc/nginx/sites-available/default`:

```
    location ~ ^/cgi-bin/.*\.fcgi$ {
            gzip off;
            include fastcgi_params;
            root /usr/lib;
            fastcgi_pass unix:/var/run/fcgiwrap.socket;
		  fastcgi_read_timeout 600;

            fastcgi_param SCRIPT_FILENAME $request_filename;
            fastcgi_param QGIS_SERVER_LOG_FILE /logs/qgisserver.log;
            fastcgi_param QGIS_SERVER_LOG_LEVEL 0;
            fastcgi_param QGIS_DEBUG 1;
    }
```

3. Cambiar permisos: `chmod g+w /var/run/fcgiwrap.socket`
4. Encender fastcgi: `service fcgiwrap start`
5. Reencender nginx: `sudo service nginx restart`

Comprobamos que GetCapabilities del QGIS Server funciona correctamente: 

http://localhost:6080/cgi-bin/qgis_mapserv.fcgi?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetCapabilities

## Configurar postgresql

1. Primero hay que instalar postgresql:
- Instalar postgresql con: `sudo apt-get install postgresql postgresql-contrib`
- Crear usuario gisadmin: `sudo -i -u gisadmin`
- Crear base de datos gis_test: `createdb gis_test`

2. Después configuramos el servicio con `pg_service.conf` para no dejar las contraseñas en los archivos de QGIS:
- Creamos el archivo `/etc/postgresql-common/pg_service.conf` con este contenido:
```
####################### gis_test
[gis_test]
host=localhost
port=5432
dbname=gis_test
user=gisadmin
password=XXX
```

- Comprobamos el acceso correcto con: `psql “service=gis_test”`

- En el caso que queremos dejar el archivo `pg_service.conf` en otro directorio, hace falta definir la varialbe `PGSERVICEFILE` en `/etc/nginx/sites-available/default`: 

`fastcgi_param PGSERVICEFILE "/var/servers/pg_service.conf";`

3. Opcional: Instalar pgrouting

- Añadir repositorio: `sudo add-apt-repository ppa:georepublic/pgrouting-unstable`
- Instalar: `sudo apt-get install postgresql-9.3-pgrouting`


## Configurar proyecto QGIS

Este paso hay que repetir para cada nuevo proyecto de QGIS que se suba al servidor. Para la descripción genérica se usa el nombre proyectoqgis tanto como nombre del fichero del proyecto como para las carpetas:

1. Subir fichero QGIS `proyectoqgis.qgs` a carpeta `/home/psig/proyectoqgis/`
2. Cambiar permisos: `chmod a+x /home/psig/proyectoqgis/proyectoqgis.qgs`
3. Crear carpeta `/usr/lib/cgi-bin/proyectoqgis/`
4. Crear enlaces simbólico a proyecto: `sudo ln -s /home/psig/proyectoqgis/proyectoqgis.qgs /usr/lib/cgi-bin/proyectoqgis/proyectoqgis.qgs`
5. Crear enlaces simbólico a script fastcgi: `sudo ln -s /usr/lib/cgi-bin/qgis_mapserv.fcgi /usr/lib/cgi-bin/proyectoqgis/qgis_mapserv.fcgi`

Ahora comprobamos el correcto funcionamiento:

1. Comprobamos que GetCapabilities del QGIS Server con este proyecto funciona correctamente:

http://localhost:6080/cgi-bin/testqgis/qgis_mapserv.fcgi?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetCapabilities

2. Comprobamos que QGIS server sirve un mapa:

http://localhost:6080/cgi-bin/testqgis/qgis_mapserv.fcgi?SERVICE=WMS&VERSION=1.3.0&SRS=EPSG:4326&REQUEST=GetMap&map=/home/psig/testqgis/countries.qgs&BBOX=-173.0000000000000000,-111.0000000000000000,187.0000000000000000,135.0000000000000000&WIDTH=550&

3. Ahora comprobamos que esta accesible desde fuera, substituyendo la IP por el dominio:

GetCapabilities: http://mapes.castelldefels.org/cgi-bin/testqgis/qgis_mapserv.fcgi?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetCapabilities

Mapa: http://mapes.castelldefels.org/cgi-bin/testqgis/qgis_mapserv.fcgi?SERVICE=WMS&VERSION=1.3.0&SRS=EPSG:4326&REQUEST=GetMap&map=/home/psig/testqgis/countries.qgs&BBOX=-173.0000000000000000,-111.0000000000000000,187.0000000000000000,135.0000000000000000&WIDTH=550&HEIGHT=500&LAYERS=countries&STYLES=,,&FORMAT=image/png 


## Ejemplo uso OpenLayers

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Test WMS MapProxy</title>
    <link rel="stylesheet" href="https://openlayers.org/en/v4.6.5/css/ol.css" type="text/css">
    <script src="https://openlayers.org/en/v4.6.5/build/ol.js"></script>
  <head>
  <body>
    <div id="map" class="map"></div>
    <script>
		var wmsLayer = new ol.layer.Tile({
			source: new ol.source.TileWMS({
				url: 'http://mapes.castelldefels.org/cgi-bin/Base_Mapa_Web/qgis_mapserv.fcgi?map=/home/psig/Base_Mapa_Web.qgs',
				params: {
					'LAYERS': 'Base_Web',
				},
				serverType: 'qgis'
			})
		});
             
		var map = new ol.Map({
			target: 'map',
			layers: [wmsLayer],
			view: new ol.View({
				center: [219484, 5053529],
				zoom: 14
			})
		});
    </script>
  </body>
</html>
```