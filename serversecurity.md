# Seguredad en servidores de mapas

## Impedir acceso por ssh con contraseña

Para impedir el acceso con contraseña, hace falta cambiar la siguiente línea en el archivo `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
```

Después reencendem sshd con `sudo service ssh restart`.

Entonces hace falta añadir la clave pública de ssh `id_rsa.pub` al archivo `.ssh/authorized_keys`.

## Cerrar puerto postgresql 5432

Para cerrar el acceso a postgresql desde Internet y permitirlo solamente desde localhost, hace falta modificar la siguiente línea en el archivo `/etc/postgresql/12/main/postgresql.conf`:

```
listen_addresses = 'localhost'
```

Después reencendem postgresql con `sudo service postgresql restart`.

