# Deploying Apache Guacamole on Raspberry Pi 3
## Step 1: Installing Raspbian lite

Download and unzip Raspbian Strech Lite:
```bash
$ wget https://downloads.raspberrypi.org/raspbian_lite_latest
$ unzip 20??-??-??-raspbian-stretch-lite.zip
```
Insert an SD card, and burn the .img file on it:
```bash
$ sudo dd bs=4M if=20??-??-??-raspbian-stretch-lite.img of=/dev/yourDevice conv=fsync
```
Mount the boot partition, and create the 'ssh' file into it:
 ```bash
$ touch ssh
```
Umount the SD card, plug in into your raspberry pi, and power it on. Make sure to connect the raspi to your LAN using the Ethernet interface.

## Step 2: Preparing the OS

Ssh into the raspi and change pi's password:
```bash
ssh pi@RaspiIpAddress
passwd
```
Resfresh apt database and apply system updates if needed:
```bash
apt update
apt full-upgrade
reboot
```
Change the hostname to something explicit:
```bash
hostnamectl set-hostname srvgcml01.home.lab
```
Edit the /etc/hosts files in order to 127.0.0.1 resolve both hostname and FQDN. You may need to log out and login again to apply every change. Example:
<pre>
127.0.0.1	srvgcml01
127.0.0.1	srvgcml01.home.lab 	
</pre>

## Step 3: Installing requirements
Create requiered directories:
```bash
mkdir -p /etc/guacamole/lib /etc/guacamole/extensions
```
Install and configure tomcat8 and mariadb-server. You need to configure mariadb manually, by following the interractive process from mysql_secure_installation.
```bash
apt install tomcat8 mariadb-server
mysql_secure_installation
```
Prepare the database from Guacamole. MySQL client will prompt you for the root password that you setuped previously. Remerber the "StrongPasswordHere" that you will define here:
```bash
mysql -u root -p
mysql> CREATE DATABASE guacamole_db;
mysql> CREATE USER 'guacamole_user'@'localhost' IDENTIFIED BY 'StrongPasswordHere';
mysql> GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole\_db.* TO 'guacamole_user'@'localhost';
mysql> FLUSH PRIVILEGES;
```
Install the JDBC driver for mariadb and link it to Guacamole configuration directory:
```bash
apt install libmysql-java
ln -s /usr/share/java/mysql-connector-java.jar /etc/guacamole/lib/
```
Download the JDBC Guacamole authentification component. Copy it to /etc/guacamole/extensions:
```bash
wget -c https://sourceforge.net/projects/guacamole/files/current/extensions/guacamole-auth-jdbc-0.9.14.tar.gz
tar xzvf guacamole-auth-jdbc-0.9.14.tar.gz
mv guacamole-auth-jdbc-0.9.14/mysql/guacamole-auth-jdbc-mysql-0.9.14.jar /etc/guacamole/extensions
```
Populate the database with the initial application data, such default admin account... One more time, insert your mariadb root password: 
```bash
cat guacamole-auth-jdbc-0.9.9/mysql/schema/*.sql | mysql -u root -p
```

## Step 4: Installing Guacamole
Install dependencies for the compilation:
```bash
apt install libcairo2-dev libossp-uuid-dev libavcodec-dev libavutil-dev libswscale-dev libfreerdp-dev libpango1.0-dev libssh2-1-dev libtelnet-dev libvncserver-dev libpulse-dev libssl-dev libvorbis-dev libwebp-dev libjpeg62-turbo-dev libpng-dev libpng16-16 git
```
Download the latest release on Github. Untar the archive.
```bash
wget -c https://github.com/apache/guacamole-server/archive/0.9.14.tar.gz
tar xfvz guacamole-server-0.9.14.tar.gz
cd guacamole-server-0.9.14
```
Configure and compile the server:
```bash
autoreconf -fi
./configure --with-init-dir=/etc/init.d
make
make install
ldconfig
systemctl enable guacd
systemctl start guacd
```

## Step 5: Create configuartion files
Create the properties file and edit it (vim, nano...). Replace "StrongPasswordHere" by the one you chose at step 3.
```bash
touch /etc/guacamole/guacamole.properties
```
<pre>
# Hostname and port of guacamole proxy
guacd-hostname: localhost
guacd-port:     4822

# MySQL properties
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: StrongPasswordHere
</pre>

Link the file in the Tomcat directory:
```bash
ln -s /etc/guacamole/ /var/lib/tomcat8/.guacamole
```
Download the webapp:
```bash
wget https://sourceforge.net/projects/guacamole/files/current/binary/guacamole-0.9.14.war
```
Move it to the proper folder, then restart Tomcat:
```bash
mv guacamole-0.9.17.war /var/lib/tomcat8/webapps/guacamole.war
systemctl restart tomcat8
```
You can now access to the application at http://ServerIP:8080/guacamole.
Default user and password is guacadmin

# Securing your installation with Apache2 reverse proxy

Install apache2 and enable ssl:
```bash
apt install apache2
a2enmod ssl
a2ensite default-ssl.conf
```

Generate self-signed SSL certificates and key. Then, secure the key:
```bash
openssl req -new -x509 -days 365 -nodes -out /etc/ssl/certs/server.crt -keyout /etc/ssl/private/server.key
chmod 440 /etc/ssl/private/server.key
```

Enable the proxy_tunnel mod:
```bash
a2enmod proxy_wstunnel
```

Edit the /etc/apache2/sites-available/default-ssl.conf file. Append/Uncomment the folowing lines INSIDE the VirtualHost configuration :
```xml
SSLCertificateFile      /etc/ssl/certs/server.crt
SSLCertificateKeyFile	 /etc/ssl/private/server.key
<Location /guacamole>
	Order allow,deny
	Allow from all
	ProxyPass http://127.0.0.1:8080/guacamole flushpackets=on
	ProxyPassReverse http://127.0.0.1:8080/guacamole
</Location>

<Location /guacamole/websocket-tunnel>
	Order allow,deny
	Allow from all
	ProxyPass ws://127.0.0.1:8080/guacamole/websocket-tunnel
	ProxyPassReverse ws://127.0.0.1:8080/guacamole/websocket-tunnel
</Location>
```
You should now be able to access Guacamole at https://ServerIP:443/guacamole.

# Step 6: Fix an issue with RDP

Create the missing folder:
```bash
mkdir /usr/lib/arm-linux-gnueabihf/freerdp/
```

Link missing librairies inside this folder:
```bash
ln -s /usr/local/lib/freerdp/guac* /usr/lib/arm-linux-gnueabihf/freerdp/
```
