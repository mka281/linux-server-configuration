# Linux Server Configuration

This project is for Udacity's [Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004).

## About the Project

### Project overview

> You will take a baseline installation of a Linux server and prepare it to host your web applications. You will secure your server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it.

### Software used for this project

* Linux
* Apache
* PostgreSQL
* Python / Flask
* mod_wsgi

### Related courses

* [Linux Command Line Basics](https://www.udacity.com/course/linux-command-line-basics--ud595)
* [Configuring Linux Web Servers](https://www.udacity.com/course/configuring-linux-web-servers--ud299)

### View the deployed version

* IP Address: [35.176.2.184](http://35.176.2.184)
* Hostname: [ec2-35-176-2-184.eu-west-2.compute.amazonaws.com](http://ec2-35-176-2-184.eu-west-2.compute.amazonaws.com)

### Steps for deployment

#### Get your server.

1.  Start a new Ubuntu Linux server instance on [Amazon Lightsail](https://lightsail.aws.amazon.com/).
2.  Follow the instructions provided to SSH into your server.

* Click to **Account page** link and download default key.
* On **Networking** section, add a new custom port 2200 with TCP protocol.
* Move key into the .ssh folder `mv ~/Downloads/LightsailDefaultPrivateKey-eu-west-2.pem ~/.ssh/udacity.pem`
* Connect to the server with this key `ssh -i ~/.ssh/udacity.pem ubuntu@YOUR_SERVER_IP`

#### Secure your server.

3.  Update all currently installed packages.

```
sudo apt-get update
sudo apt-get upgrade
```

4.  Change the SSH port from 22 to 2200. Make sure to configure the Lightsail firewall to allow it.

* Go to config file with `sudo nano /etc/ssh/sshd_config`
* Change Port line from 22 to 2200 and save the file.

5.  Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
    Deny incoming connections by default with `sudo ufw default deny incoming`
    Allow outgoing connections by default with `sudo ufw default allow outgoing`
    Define ports for incoming connections

```
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
```

View ufw status with `sudo ufw status`

#### Give grader access.

6.  Create a new user account named grader.

* `sudo adduser grader`

7.  Give grader the permission to sudo.

* Open a new file with `sudo nano /etc/sudoers.d/grader`
* Type `grader ALL=(ALL) NOPASSWD:ALL` and save.

8.  Create an SSH key pair for grader using the ssh-keygen tool.

* Generate ssh key on your local machine with `ssh-keygen -f ~/.ssh/udacity.rsa`
* Copy the generated key with `cat ~/.ssh/udacity.rsa.pub`
* On the AWS server, change user to grader with `su - grader`
* Create `.ssh` folder `mkdir .ssh`
* Open a new file with `nano .ssh/authorized_keys`
* Paste the key you copied here and save the file.
* Give necessary permissions with

```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

* Change the owner from root to grader with: `sudo chown -R grader:grader .ssh`

#### Prepare to deploy your project.

9.  Configure the local timezone to UTC. `sudo dpkg-reconfigure tzdata.`

10. Install and configure Apache to serve a Python mod_wsgi application.

* Install Apache `sudo apt-get install apache2`
* Install mod_wsgi `sudo apt-get install libapache2-mod-wsgi python-dev` and enable it with `sudo a2enmod wsgi`
* Start apache server with `sudo service apache2 start`

11. Install and configure PostgreSQL:

* Install necessary Python packages `sudo apt-get install libpq-dev python-dev`
* Install PostgreSQL `sudo apt-get install postgresql postgresql-contrib`
* Change user to postgres `sudo su - postgres`

- Create a new database user named catalog that has limited permissions to your catalog application database.

```
CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog with OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
$ exit
```

* Do not allow remote connections. Check **pg_hba.conf** file with `sudo nano /etc/postgresql/9.3/main/pg_hba.conf`. Make sure the output without the comment lines is the same with this:

```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

12. Install git. `sudo apt-get install git`

#### Deploy the Item Catalog project.

13. Clone and setup your Item Catalog project from the Github repository you created earlier in this Nanodegree program.

* Create a new `catalog` directory within `/var/www/`

a. Clone the repo with `git clone https://github.com/mka281/item-catalog.git catalog`

* Change the name of application.py to **init**.py `mv application.py __init__.py`
* Define the directory of `client_secrets.json` with `/var/www/catalog/password/client_secrets.json`
* Change `sqlite:///itemcatalog.db` into the `postgresql://catalog:catalog@localhost/catalog`

b. Create a new file named 'catalog.wsgi' to serve our application over mod_wsgi. `sudo nano catalog.wsgi` and make sure application.secret_key is the same with your app's secret key.

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'secret_key'
```

c. Setup virtual environment

* Install pip package manager `sudo apt-get install python-pip`
* Install virtualenv `sudo pip install virtualenv`
* Make sure you are in the `/var/www/catalog` directory and create a new virtual environment with `sudo virtualenv venv`
* Activate created virtual environment `source venv/bin/activate`
* Give necessary permissions to the virtual environment `sudo chmod -R 777 venv`
* Install flask `pip instal flask`
* Install necessary dependencies for the project `pip install httplib2 request oauth2client sqlalchemy psycopg2`

14. Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser.

* Configure and enable a new virtual host:
* Create a config file: `sudo nano /etc/apache2/sites-available/catalog.conf`
* Paste this into the file and save

```
<VirtualHost *:80>
    ServerName YOUR.AWS.IP.ADDRESS
    ServerAlias ec2-YOUR-AWS-IP-ADDRESS.us-west-2.compute.amazonaws.com
    ServerAdmin admin@YOUR.AWS.IP.ADDRESS
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

* Enable new virtual host `sudo a2ensite catalog`
* Restart the Apache server with `sudo service apache2 restart`

## Additional Resources

* How To Deploy a Flask Application on an Ubuntu VPS [https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps]
* How To Secure PostgreSQL on an Ubuntu VPS [https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps]
* Thanks to [iliketomatoes](https://github.com/iliketomatoes) for a helpful README.
