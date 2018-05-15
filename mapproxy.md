# Instalación y configuración MapProxy con Gunicorn

Basado en manual de instalación oficial de [MapProxy](https://mapproxy.org/docs/1.11.0/install.html)

1. [Instalar MapProxy](#instalar-mapproxy)
2. [Configurar MapProxy](#configurar-mapproxy)
3. [Comprobar MapProxy](#comprobar-mapproxy)
4. [Actualizar MapProxy](#actualizar-mapproxy)
5. [Gunicorn HTTP Server](#gunicorn-http-server)
6. [Ejemplo uso OpenLayers](#ejemplo-uso-openlayers)

## Instalar MapProxy

```
sudo apt-get install python-setuptools
sudo easy_install MapProxy
mapproxy-util --version
mapproxy-util create -t base-config /home/psig/mapproxy
```

Nota: Si hay más programas de python funcionando en el servidor, entonces es recomendado instalarlo dentro de un [entorno virtual](https://mapproxy.org/docs/1.11.0/install.html#create-a-new-virtual-environment).

## Configurar MapProxy

Toda la configuración de MapProxy esta en el archivo `/home/psig/mapproxy/mapproxy.yaml`:

```yaml
services:
  demo:
  wms:
    srs: ['EPSG:4326', 'EPSG:3857', 'EPSG:25831']
    bbox_srs: ['EPSG:4326', 'EPSG:3857', 'EPSG:25831']
    md:
      title: Castefa MapProxy WMS Proxy
      abstract: MapProxy WMS Proxy for Castefa
      online_resource: http://www.castelldefels.org
      contact:
          organization: PSIG
          email: info@psig.es
      access_constraints:
          This service is for internal use only.
      fees: 'None'

layers:
  - name: Base_Mapa_Web
    title: base mapa web
    sources: [sentits_circulacio,base_web]
  - name: Base_Web
    title: SSA WMS - Base Web
    sources: [base_web]
  - name: Sentits_circulacio
    title: Sentits circulacio
    sources: [sentits_circulacio]

caches:
  base_web:
    grids: [webmercator]
    sources: [base_web]
  sentits_circulacio:
    grids: [webmercator]
    sources: [sentits_circulacio]
    meta_size: [4,4]
    meta_buffer: 100

sources:
  base_web:
    type: wms
    req:
      url: http://localhost:6969/cgi-bin/Base_Mapa_Web/qgis_mapserv.fcgi
      layers: Base_Web
      transparent: true
  sentits_circulacio:
    type: wms
    req:
      url: http://localhost:6969/cgi-bin/Base_Mapa_Web/qgis_mapserv.fcgi
      layers: Sentits circulació
      transparent: true

grids:
    webmercator:
        base: GLOBAL_WEBMERCATOR
        res: [50,25,10,5,2,1,0.5,0.25,0.125]

globals:
```

Nota: Para añadir más capas, hay que añadir los parámetros `layers`, `caches`, `sources` basado en la documentación de [MapProxy](https://mapproxy.org/docs/1.11.0/configuration.html).

## Comprobar MapProxy

Para probar, lo arrancamos con: 

`mapproxy-util serve-develop mapproxy.yaml`

Después se puede acceder a través de una interfaz web de prueba en http://localhost:6969/mapproxy

## Actualizar MapProxy

En caso de actualizar capas del proyecto QGIS hace falta adaptar la configuración de MapProxy. Para eso editamos el archivo `/home/psig/mapproxy/mapproxy.yaml`.

Después hay que borrar los tiles guardados por MapProxy en `/home/psig/mapproxy/cache_data`. Los tiles de cada capa se guardan en una carpeta que contiene el nombre de la capa (o del grupo) y la proyección (por ejemplo `EPSG3857`). Borra la capa con `rm -rf nombre_carpeta` o por FTP.

## Gunicorn HTTP Server

MapProxy contiene un servidor HTTP sencillo, para entornos de producción esta recomendado utilizar un servidor HTTP más estable. Utilizaremos Gunicorn, un servidor HTTP escrito en python.

### Instalar

`sudo apt-get install python-eventlet gunicorn`

### Crear webservice

Creamos el archivo `/home/psig/mapproxy/config.py`:

```
from mapproxy.wsgiapp import make_wsgi_app
application = make_wsgi_app(r'/home/psig/mapproxy/mapproxy.yaml', reloader=True)
```

### Probar Gunicorn

Podemos probar el correcto funcionamiento de Gunicorn con: 

`gunicorn -k eventlet -w 4 -b :8080 config:application –no-sendf`

### Crear servicio de arranque

Creamos el archivo `/etc/init/mapproxy.conf`:

```
start on runlevel [2345]
stop on runlevel [!2345]

respawn

setuid mapproxy
setgid mapproxy

chdir /home/psig/mapproxy

exec gunicorn -k eventlet -w 8 -b :8080 \
	--no-sendfile \
	config:application \
	>>/var/log/mapproxy/gunicorn.log 2>&1
```

Se puede comprobar el correcto funcionamiento en http://localhost:8080 

### Crear proxy nginx

Añadimos la siguiente configuración a `/etc/nginx/available-sites/default` o similar:

```
location /mapproxy {
	proxy_pass http://localhost:8080;
	proxy_set_header Host $http_host;
	proxy_set_header X-Script-Name /mapproxy;
}
```

Ahora la interfaz de prueba esta accesible en http://localhost/mapproxy 

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
				url: 'http://mapes.castelldefels.org/mapproxy/service',
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