This repository contains Seafile server package (seafile-server-8.0.7-buster-armv6l.tar.gz) for installing on Raspberry Pi Zero (armv6l). Should also work on all newer Raspberry Pi, but not optimal as this only uses armv6l features. The package was built using QEMU armv6l kernel 4.19 Raspbian Buster in [dockerpi](https://github.com/lukechilds/dockerpi).

## Download

- The latest **stable** rpi version is suppose to be found [here](https://github.com/haiwen/seafile-rpi/releases/latest). But they did not have versions for armv6l, so compiled a version for myself, I think it took like 4 hours on my old laptop.

## Installation Instructions

These are just from my notes in setting up Seafile, best to doublecheck with official instructions.

### Manual and Guides

- [Build Seafile server](https://manual.seafile.com/build_seafile/rpi/)
- [Deploy Seafile server](https://manual.seafile.com/deploy/)

### Okay here goes.

**[Setup MariaDB on Debian](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-debian-10)**
```zsh
# setup sql on pi
sudo apt update
sudo apt install mariadb-server
# secure sql server
sudo mysql_secure_installation
# answer menu:
Enter - no database root password since 1st time
y - set up root password - makes it easyier to setup, just delete mysql root user after everything is working fine
y - remove anonymous user
y - disallow root login remotely
y - remove test dataset
y - reload privilege tables

# change sql server authentication to password not plugin
sudo mysql
# add user, change 'user' and 'password' to prefered values
MariaDB [(none)]> GRANT ALL ON *.* TO 'admin'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> exit
sudo systemctl status mariadb # check running service is active

mysqladmin -u admin -p version # check if added 'admin' user works
# how to change a user password:
# systemctl stop mariadb
# SELECT user FROM mysql.user; # list all users
# alter user 'root@localhost' identified by 'new_password';
# REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'user_name'@'localhost';
# DROP USER 'user_name'@'localhost';
```

**Setup seafile on pi**
```zsh
sudo apt-get install python3 python3-setuptools python3-pip default-libmysqlclient-dev -y
sudo apt install python3-pymysql
sudo pip3 install --timeout=3600 Pillow pylibmc captcha jinja2 sqlalchemy==1.4.3 django-pylibmc django-simple-captcha python3-ldap mysqlclient

# create program directory
mkdir /opt/seafile
cd /opt/seafile
adduser seafile # create user/password
chown -R seafile: /opt/seafile

su seafile # change to user 'seafile'
tar -xfv seafile-server-8.0.7-buster-armv6l.tar.gz
cd seafile-server-8.0.7
./setup-seafile-mysql.sh
# menu responces
server name: choose_a_server_name
server ip: my_domain.com
port 8082
1 - create new
Enter - default: localhost
Enter - default mysql port: 3306
Enter - default user: seafile
setup password for mysql user seafile

# if you did not create a mysql root user, run these commands:
#create database `ccnet_db` character set = 'utf8';
#create database `seafile_db` character set = 'utf8';
#create database `seahub_db` character set = 'utf8';
#create user 'seafile'@'localhost' identified by 'seafile';
#GRANT ALL PRIVILEGES ON `ccnet_db`.* to `seafile`@localhost;
#GRANT ALL PRIVILEGES ON `seafile_db`.* to `seafile`@localhost;
#GRANT ALL PRIVILEGES ON `seahub_db`.* to `seafile`@localhost;

# Change the bind to "0.0.0.0:8000"
vi /opt/seafile/conf/gunicorn.conf.py

# Add port 8000 (i.e., SERVICE_URL = http://1.2.3.4:8000/)
vi /opt/seafile/conf/ccnet.conf

# if LC_ALL is not set in ENV message
sudo dpkg-reconfigure locales # select en_US.UTF-8

cd /opt/seafile-server-latest
./seafile.sh start

# create symbolic link to fix error in seahub.sh export
# there was a python2.7 folder, so linked it to correct python3.6
cd /opt/seafile/seafile-server-8.0.7/seafile/lib
mkdir python3.6
ln -s ./python2.7/site-packages ./python3.6/site-packages

./seahub.sh start
# setup account on first use
my_email my_password

# if seahub error Pillow: libopenjp2.so.7: cannot open shared object
sudo apt install libopenjp2-7
```

**Setup nginx with ssl**
```zsh
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx

touch /etc/nginx/sites-available/seafile.conf
rm /etc/nginx/sites-enabled/default
rm /etc/nginx/sites-available/default
ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf

# edit seafile.conf as shown in https://manual.seafile.com/deploy/https_with_nginx/
nano /etc/nginx/sites-available/seafile.conf

nginx -t
nginx -s reload

# setup certbot
https://certbot.eff.org/lets-encrypt/pip-nginx

sudo apt install python3-venv libaugeas0
sudo apt-get remove certbot

sudo python3 -m venv /opt/certbot/
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot certbot-nginx
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot
sudo certbot certonly --nginx

# setup automatic cert renewal
echo "0 0,12 * * * root /opt/certbot/bin/python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew -q" | sudo tee -a /etc/crontab > /dev/null

# monthly upgrade certbot:
sudo /opt/certbot/bin/pip install --upgrade certbot certbot-nginx
# if error upgrading
sudo rm -rf /opt/certbot
# repeat installation

# edit seafile.conf to add ssl section as show on (also update server name): https://manual.seafile.com/deploy/https_with_nginx/
vi /opt/seafile/conf/seafile.conf
# also add to [fileserver] to force access only via nginx, host = 127.0.0.1  ## default port 0.0.0.0

sudo nginx -t
sudo service nginx reload

# update ccnet.conf to use https, SERVICE_URL = https://seafile.example.com
vi /opt/seafile/conf/ccnet.conf

# update seahub_settings.py to use https, FILE_SERVER_ROOT = 'https://seafile.example.com/seafhttp'
vi /opt/seafile/conf/seahub_settings.py

# Optional more secure nginx ssl configuration
# edit nginx seafile.conf
# HSTS for protection against man-in-the-middle-attacks
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
# DH parameters for Diffie-Hellman key exchange
ssl_dhparam /etc/nginx/dhparam.pem;

# generate dhparam.pem, takes a long time
openssl dhparam 2048 > /etc/nginx/dhparam.pem  # Generates DH parameter of length 2048 bits
# use nohup to keep process running while disconnecting from ssh
nohup long-running-command &

# sudo openssl req $@ -new -x509 -days 730 -nodes -out /etc/nginx/cert.pem -keyout /etc/nginx/cert.key

# generate example nginx ssl config at https://ssl-config.mozilla.org/#server=nginx&version=1.14.2&config=modern&openssl=1.1.1d&guideline=5.6
--------
# generated 2021-10-24, Mozilla Guideline v5.6, nginx 1.14.2, OpenSSL 1.1.1d, modern configuration
# https://ssl-config.mozilla.org/#server=nginx&version=1.14.2&config=modern&openssl=1.1.1d&guideline=5.6
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    ssl_certificate /path/to/signed_cert_plus_intermediates;
    ssl_certificate_key /path/to/private_key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    # modern configuration
    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;

    # HSTS (ngx_http_headers_module is required) (63072000 seconds)
    add_header Strict-Transport-Security "max-age=63072000" always;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    # verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;

    # replace with the IP address of your resolver
    resolver 127.0.0.1;
}

# check ssl configuration https://www.ssllabs.com/ssltest/analyze.html
```

**Set seafile/seahub service for startup**
https://manual.seafile.com/deploy/start_seafile_at_system_bootup/
```zsh
sudo vim /etc/systemd/system/seafile.service
# start file seafile.service
[Unit]
Description=Seafile
After=network.target mysql.service
[Service]
Type=forking
ExecStart=${seafile_dir}/seafile-server-latest/seafile.sh start
ExecStop=${seafile_dir}/seafile-server-latest/seafile.sh stop
LimitNOFILE=infinity
User=seafile
Group=seafile
[Install]
WantedBy=multi-user.target
# end file seafile.service

# set seahub service for startup
sudo vim /etc/systemd/system/seahub.service
# start file seahub.service
[Unit]
Description=Seahub
After=network.target seafile.service
[Service]
Type=forking
# change start to start-fastcgi if you want to run fastcgi
ExecStart=${seafile_dir}/seafile-server-latest/seahub.sh start
ExecStop=${seafile_dir}/seafile-server-latest/seahub.sh stop
User=seafile
Group=seafile
[Install]
WantedBy=multi-user.target
# end file seahub.service

sudo systemctl enable seafile.service
sudo systemctl enable seahub.service
```

## Contributors

Thanks to all.
See [CONTRIBUTORS](https://github.com/haiwen/seafile-rpi/graphs/contributors).
