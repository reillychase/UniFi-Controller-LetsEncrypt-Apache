# UniFi-Controller-LetsEncrypt-Apache
A guide to obtain a free, valid SSL for UniFi Controller. This method actually uses an SSL'd Apache proxy rather than dealing with the complexity of updating UniFi's built in SSL.

# Fresh Debian 8 Instructions:

## Install UniFi Controller
'''
sudo apt-get update -y
sudo apt-get upgrade -y
echo "deb http://www.ubnt.com/downloads/unifi/debian unifi5 ubiquiti" > /etc/apt/sources.list.d/ubnt.list 
apt-key adv --keyserver keyserver.ubuntu.com --recv C0A52C50
apt-get update -y
apt-get install unifi -y
'''

## Install Let's Encrypt certbot
echo 'deb http://ftp.debian.org/debian jessie-backports main' | sudo tee /etc/apt/sources.list.d/backports.list
sudo apt-get update
sudo apt-get install python-certbot-apache -t jessie-backports

## Run the Certbot wizard, specify your domain like unifi.example.com, and email address
certbot --apache

## Add cronjob to auto renew cert every Monday at 2:30am
sudo crontab -e
30 2 * * 1 /usr/bin/certbot renew >> /var/log/le-renew.log

## Add modules to Apache for Proxying HTTP/HTTPS to 8080 and 8443
mkdir /var/www/unifi
a2enmod proxy
a2enmod proxy_http

service apache2 restart

nano /etc/apache2/sites-enabled/000-default.conf

## Example of 000-default.conf, change unifi.example.com to your site:

<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	ServerName unifi.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/unifi

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
	
	<Directory /var/www/unifi>
		Options None
		AllowOverride None
		Require all granted
	</Directory>
</VirtualHost>

## Example of 000-default-le-ssl.conf, change unifi.example.com to your site:

<IfModule mod_ssl.c>
<VirtualHost *:443>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	ServerName unifi.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/unifi
	SSLProxyEngine On
	SSLProxyCheckPeerCN off
	SSLProxyCheckPeerName off
	ProxyPreserveHost On
	ProxyPass / https://unifi.example.com:8443/	

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

	
SSLCertificateFile /etc/letsencrypt/live/unifi.example.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/unifi.example.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf


</VirtualHost>

</IfModule>

## Restart Apache for changes to take effect
service apache2 restart
