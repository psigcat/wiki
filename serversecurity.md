# Seguredad en servidores de mapas

## Impedir acceso por ssh con contraseña

Para impedir el acceso con contraseña, hace falta cambiar la siguiente línea en el archivo `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
```

Después reencendem sshd con `sudo service ssh restart`.

Entonces hace falta añadir la clave pública de ssh `id_rsa.pub` al archivo `.ssh/authorized_keys`.

### Configurar pgadmin4 para acceder con SSH Tunnel

Para que pgadmin4 accede por ssh hace falta añadir usuario y la llave privada al campo *Identity file* en la pestaña SSH tunnel:

![](images/pgadmin4.png)

### Configurar Qgis para acceder con SSH Tunnel

Aún nos falta comprobar si existe una posiblidad más sencilla de acceder desde Qgis usando un tunnel ssh que descrito en este documento: https://tripgis.wordpress.com/2015/02/19/creating-a-remote-postgis-backend-to-use-with-qgis/

## Cerrar puerto postgresql 5432

Para cerrar el acceso a postgresql desde Internet y permitirlo solamente desde localhost, hace falta modificar la siguiente línea en el archivo `/etc/postgresql/12/main/postgresql.conf`:

```
listen_addresses = 'localhost'
```

Después reencendem postgresql con `sudo service postgresql restart`.

