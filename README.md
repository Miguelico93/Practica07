# Practica07

## Wordpress con Vagrant.

La infraestructura estará formada por:
* Un balanceador de carga, implementado con un  Apache HTTP Server configurado como proxy inverso.
* Una capa de front-end, formada por dos servidores web con Apache HTTP Server.
* Una capa de back-end, formada por un servidor MySQL.

Las direcciones IPs serán las siguientes:
* Balanceador: 192.168.33.10   
* Frontal Web 1: 192.168.33.11
* Frontal Web 2: 192.168.33.12
* Servidor MySQL: 192.168.33.13

## Sincronización del contenido estático en el Front-End:

Para no tener problemas a la hora de instalar temas o plugins para wordpress vamos a hacer una configuración:

Utilizar almacenamiento compartido por NFS del directorio ```/var/www/html/wp-content``` entre todos los servidores de la capa de front-end.

Así podremos evitar que cuando hagamos una instalación en uno de nuestros frontales, los demás no tegan constancia de que ha sido instalado.

### Configuración:

Podemos utilizar NFS para que los servidores de la capa de front-end compartan el directorio ```/var/www/html/wp-content```. En nuestro caso un frontal hará de servidor NFS y el otro de cliente NFS. El servidor NFS compartirá el directorio  ```/var/www/html/wp-content``` y el cliente podrá montar este directorio en su sistema de ficheros.

Configuración de las IPs:

* Servidor NFS: 192.168.33.11
* Cliente NFS: 192.168.33.12

#### Instalación de paquetes:

Servidor NFS:
```
sudo apt-get update
sudo apt-get install nfs-kernel-server
```

Clientes NFS:
```
sudo apt-get update
sudo apt-get install nfs-common
```

#### Exportación del directorio en el servidor NFS

Cambiar permisos del directorio que vamos a compartir:
```
sudo chown nobody:nogroup /var/www/html/wp-content
```
Editamos el archivo ``/etc/exports``:
```
sudo nano /etc/exports
```
Añadimos la siguiente linea:
```
/var/www/html/wp-content      192.168.33.12(rw,sync,no_root_squash,no_subtree_check)
```
### Reiniciar el servicio NFS:
```
sudo /etc/init.d/nfs-kernel-server restart
```
### Creamos el punto de montaje en el cliente NFS:
```
sudo mount 192.168.33.11:/var/www/html/wp-content /var/www/html/wp-content
```
Una vez hecho esto comprobamos con df -h que le punto de montaje aparece en el listado.
```
$ df -h

udev                                    490M     0  490M   0% /dev
tmpfs                                   100M  3.1M   97M   4% /run
/dev/sda1                               9.7G  1.1G  8.6G  12% /
tmpfs                                   497M     0  497M   0% /dev/shm
tmpfs                                   5.0M     0  5.0M   0% /run/lock
tmpfs                                   497M     0  497M   0% /sys/fs/cgroup
192.168.33.11:/var/www/html/wp-content  9.7G  1.1G  8.6G  12% /var/www/html/wp-content
tmpfs                                   100M     0  100M   0% /run/user/1000
```
### Editamos el archivo /etc/fstab en el cliente NFS
Editamos el archivo /etc/fstab para que al iniciar la máquina se monte automáticamente el directorio compartido por NFS.
```
sudo nano /etc/fstab
```
Añadimos la siguiente línea:
```
192.168.33.11:/var/www/html/wp-content /var/www/html/wp-content  nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0
```

# Vagrant.file
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/bionic64"
  
  # Apache HTTP Servidor 1
  config.vm.define "web1" do |app|
    app.vm.hostname = "web1"
    app.vm.network "private_network", ip: "192.168.33.10"
    app.vm.provision "shell", path: "provision/script-apache.sh"
  end

  # Apache HTTP Servidor 2
  config.vm.define "web2" do |app|
    app.vm.hostname = "web2"
    app.vm.network "private_network", ip: "192.168.33.11"
    app.vm.provision "shell", path: "provision/script-apache.sh"
  end

  # Servidor MySQL
  config.vm.define "db" do |app|
    app.vm.hostname = "db"
    app.vm.network "private_network", ip: "192.168.33.12"
    app.vm.provision "shell", path: "provision/script-sql.sh"
  end

  # Balanceador
  config.vm.define "balanceador" do |app|
    app.vm.hostname = "balan"
    app.vm.network "private_network", ip: "192.168.33.2"
    app.vm.provision "shell", path: "provision/script-balanceador.sh"
  end

end
```
# 000-Default.conf
```
<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	#ServerName www.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html

	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	# For most configuration files from conf-available/, which are
	# enabled or disabled at a global level, it is possible to
	# include a line for only one particular virtual host. For example the
	# following line enables the CGI configuration for this host only
	# after it has been globally disabled with "a2disconf".
	#Include conf-available/serve-cgi-bin.conf

	<Proxy balancer://mycluster>
        # Server 1
        BalancerMember http://192.168.33.10

        # Server 2
        BalancerMember http://192.168.33.11
    </Proxy>

    ProxyPass / balancer://mycluster/
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```
# Scripts:

## Apache:
```
#!/bin/bash

# Actualización e instalación de Apache
apt-get update
apt-get install -y apache2
apt-get install -y php libapache2-mod-php php-mysql
sudo /etc/init.d/apache2 restart

# Instalación de wget
apt-get install -y wget
cd /tmp
rm -rf latest.tar.gz
rm -rf wordpress
wget https://wordpress.org/latest.tar.gz

# Descomprimir el archivo
tar -xzvf *.tar.gz

# Definimos las variables
NOMBRE_BD=wordpress
NOMBRE_USU=word
PASSWD_BD=123456
SERVER=192.168.33.12

# Credenciales de Wordpress
cd wordpress 
mv wp-config-sample.php wp-config.php
sed -i "s/database_name_here/$NOMBRE_BD/" wp-config.php
sed -i "s/username_here/$NOMBRE_USU/" wp-config.php
sed -i "s/password_here/$PASSWD_BD/" wp-config.php
sed -i "s/localhost/$SERVER/" wp-config.php


# Copiamos la carpeta Wordpress y le damos permisos de www-data
cp . /var/www/html -R
chown www-data:www-data /var/www/html/* -R

# Borramos el index.html
rm -rf /var/www/html/index.html
```
## Balanceador:
```
#!/bin/bash

#Actualización y upgrade
apt-get update
apt-get install -y apache2

# Activamos los modulos
a2enmod proxy
a2enmod proxy_http
a2enmod proxy_ajp
a2enmod rewrite
a2enmod deflate
a2enmod headers
a2enmod proxy_balancer
a2enmod proxy_connect
a2enmod proxy_html
a2enmod lbmethod_byrequests

# Eliminamos 000-DEAFULT.CONF
rm -rf /etc/apache2/sites-enabled/000-default.conf
cp /vagrant/archivos/000-default.conf /etc/apache2/sites-enabled/

# Reinciamos el servicio
/etc/init.d/apache2 restart
```
## MySQL:
```
#!/bin/bash

# Actualización y upgrade
apt-get update
apt-get -y install debconf-utils

#Configuración del root
DB_ROOT_PASSWD=123456
debconf-set-selections <<< "mysql-server mysql-server/root_password password $DB_ROOT_PASSWD"
debconf-set-selections <<< "mysql-server mysql-server/root_password_again password $DB_ROOT_PASSWD"

#Instalación MySQL
apt-get install -y mysql-server
sed -i -e 's/127.0.0.1/0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
/etc/init.d/mysql restart

mysql -uroot mysql -p$DB_ROOT_PASSWD <<< "GRANT ALL PRIVILEGES ON *.* TO root@'%' IDENTIFIED BY '$DB_ROOT_PASSWD'; FLUSH PRIVILEGES;"
mysql -uroot mysql -p$DB_ROOT_PASSWD < /vagrant/provision/script/wordpress.sql
```
## Cliente NFS:
```
#!/bin/bash

# Incluir provision-for-apache
source /vagrant/provision/provision-for-apache-sh

# Instalación cliente NFS
apt-get update
apt-get install nfs-common

# Montar wp-content
mount 192.168.33.11:/var/www/html/wp-content /var/www/html/wp-content
```
## Servidor NFS:
```
#!/bin/bash
set -x

# Incluir provision-for-apache.sh
source /vagrant/provision/provision-for-apache.sh

# Instalación nfs-server
apt-get update
apt-get install nfs-kernel-server

# Cambiar permisos
chown nobody:nogroup /var/www/html/wp-content

# Copiar y exportar
cp /vagrant/config/exports /etc/ -f

# Reinciar servicio NFS
/etc/init.d/nfs-kernel-server Restart
```
## Wordpress SQL:
```
DROP DATABASE IF EXISTS wordpress;
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;

USE mysql;
CREATE USER 'word'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON wordpress.* TO word@'%' IDENTIFIED BY '123456'; 
FLUSH PRIVILEGES;
````
