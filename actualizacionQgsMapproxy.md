# Actualización mapas GIS en servidor

La actualización de los mapas en nuestros servidores GIS pasa por 3 partes:

1. [Publicar proyecto actualizado](#1-publicar-mapa-actualizado)
2. [Parsear proyecto Qgis](#2-parsear-proyecto-qgis)
3. [Regenerar tiles MapProxy](#3-regenerar-tiles-mapproxy)

## 1. Publicar mapa actualizado

Nuestros mapas están en la carpeta `/var/servers/[PROYECTO]/maps`, se trata de subir el proyecto .qgs actualizado a esta carpeta y sobreescribir el archivo existente.

## 2. Parsear proyecto Qgis

Usamos el script [qgs_parser](https://github.com/geraldo/qgs_parser) para parsear el archivo `.qgs` y generar un archivo de configuración `.json`. Este archivo contiene la configuración de capas para el visor web, tanto visibilidad, identificación como los campos de features a mostrar.

El script se arranca con: 
`cd /var/servers/[PROYECTO]/maps/`
`./parse.sh [PROYECTO].qgs`

El json resultando se guarda en la carpeta `/var/servers/[PROYECTO]/web/js/data/[PROYECTO].qgs.json`

## 3. Regenerar tiles MapProxy

Como último regeneramos los tiles de [MapProxy](https://mapproxy.org/).

La instalación de MapProxy tenemos en `/opt/mapproxy/[PROYECTO]/`, los tiles generados en la carpeta `/opt/mapproxy/[PROYECTO]/cache_data`.

El seeding completo arrancamos con:

`sudo mapproxy-seed -f mapproxy.yaml -s seed_[PROEYCTO].yaml -c 8 --seed ALL > seed_[PROYECTO].log`

Si solamente queremos regenerar los tiles de una capa específica o un grupo de capas, lo arrancamos con:

`sudo mapproxy-seed -f mapproxy.yaml -s seed_[PROEYCTO].yaml -c 8 --seed [PROYECTO]_[CAPA/GRUPO] > seed_[PROYECTO]_[CAPA/GRUPO].log`

Donde `[CAPA/GRUPO]` es el nombre de la capa definido en `/opt/mapproxy/[PROYECTO]/mapproxy.yaml`.