# Instalar y configurar MapProxy

Basado en manual de instalación oficial de [MapProxy](https://mapproxy.org/docs/1.11.0/install.html)

- [Instalar MapProxy](instalar-mapproxy)
- [Configurar MapProxy](configurar-mapproxy)
- [Probar MapProxy](probar-mapproxy)
- [Gunicorn HTTP Server](gunicorn-http-server)

## Instalar MapProxy

`sudo apt-get install python-setuptools
sudo easy_install MapProxy
mapproxy-util --version
mapproxy-util create -t base-config /home/psig/mapproxy`

Nota: Si hay más programas de python funcionando en el servidor, entonces es recomendado instalarlo dentro de un entorno virtual: https://mapproxy.org/docs/1.11.0/install.html#create-a-new-virtual-environment

## Configurar MapProxy

Toda la configuración de MapProxy esta en el archivo `/home/psig/mapproxy/mapproxy.yaml`.

`services:
  demo:
  wms:
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
  - name: Base_Web
    title: SSA WMS - Base Web
    sources: [base_web]

caches:
  base_web:
    grids: [webmercator]
    sources: [base_web]

sources:
  ssa_layer_tm:
    type: wms
    req:
      url: http://localhost:6969/cgi-bin/Base_Mapa_Web/qgis_mapserv.fcgi
      layers: Base_Web

grids:
    webmercator:
        base: GLOBAL_WEBMERCATOR

globals:`

Nota: Para añadir más capas, hay que añadir los parámetros `layers`, `caches`, `sources` basado en la documentación de MapProxy: https://mapproxy.org/docs/1.11.0/configuration.html 

## Probar MapProxy

Para probar, lo arrancamos con: `mapproxy-util serve-develop mapproxy.yaml`

Después se puede acceder a través de una interfaz web de prueba en http://localhost:6969/mapproxy

## Gunicorn HTTP Server

MapProxy contiene un servidor HTTP sencillo, para entornos de producción esta recomendado utilizar un servidor HTTP más estable. Utilizaremos Gunicorn, un servidor HTTP escrito en python.

1. Instalar:
`sudo apt-get install python-eventlet gunicorn`

2. Crear webservice:

Creamos el archivo `/home/psig/mapproxy/config.py`:

`from mapproxy.wsgiapp import make_wsgi_app
application = make_wsgi_app(r'/home/psig/mapproxy/mapproxy.yaml')`

3. Probar Gunicorn:

Podemos probar el correcto funcionamiento de Gunicorn con: 

`gunicorn -k eventlet -w 4 -b :8080 config:application –no-sendf`

4. Crear servicio de arranque:

Creamos el archivo `/etc/init/mapproxy.conf`:

`start on runlevel [2345]
stop on runlevel [!2345]

respawn

setuid mapproxy
setgid mapproxy

chdir /home/psig/mapproxy

exec gunicorn -k eventlet -w 8 -b :8080 \
    --no-sendfile \
    application \
    >>/var/log/mapproxy/gunicorn.log 2>&1`

Se puede comprobar el correcto funcionamiento en http://localhost:8080 

5. Crear proxy nginx:

`location /mapproxy {
	proxy_pass http://localhost:8080;
    proxy_set_header Host $http_host;
    proxy_set_header X-Script-Name /mapproxy;
}`

Ahora la interfaz de prueba esta accesible en http://localhost/mapproxy 
