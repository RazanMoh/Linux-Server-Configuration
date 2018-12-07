# Linux Server Configuration

## Introduction

A project for a setup and configure a Linux (Ubuntu) web server using Amazon AWS. The server must be secure and serve the Item Catalog website. This project is a part of the Udacity's Full Stack Web Developer Nanodegree.

Public IP Address: 18.217.94.162
SSH Port: 2200 
SSH login as Grader: ssh -v -i ~/.ssh/LightsailDefault.pem grader@18.184.151.245 -p 2200
Application URL: http://ec2-18-184-151-245.eu-central-1.compute.amazonaws.com|

## Get Started

#### ***Server Setup:***
#### ***Step 1: Start a new Ubuntu Linux server instance on Amazon Lightsail***

1.	Login into Amazon Lightsail using an Amazon Web Services account.
2.	Once you're logged in, click on Create an instance.
3.	Choose instance image, OS Only and Ubuntu 16.04 LTS.
4.	Choose an instance plan
5.	Give your instance a hostname
6.	Click the Create button to create the instance
7.	Wait for the instance to start up
8.	Login to the instance via command line

```bash
ssh -i ~/.ssh/ LightsailDefault.pem ubuntu@[YOUR.PUBLIC.IP.ADDRESS]
```
#### ***Step 2: Install updates/upgrades and fix timezone***
1.	Login to virtual machine (VM)
2.	Update all currently installed packages:

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install finger
sudo apt-get install ntp
```
3.	Configure the timezone to UTC
```bash
sudo dpkg-reconfigure tzdata
```
#### ***Step 3: Create a new user named grader and grant this user sudo permissions***
1. Add new user 'grader' and give it sudo permissions
```bash
sudo adduser grader
sudo nano /etc/sudoers.d/grader
```
- Type grader ALL=(ALL:ALL) ALL in the sudoer.d/grader file you opened. Save it
- Run the command sudo nano /etc/hosts 
- In order to prevent the "sudo: unable to resolve host" error add 127.0.1.1 ip-[private]-[ip]-[address]

2. SSH login for grader
```bash
sudo mkdir /home/grader/.ssh
sudo touch /home/grader/.ssh/authorized_keys
sudo cp /root/.ssh/authorized_keys /home/grader/.ssh/authorized_keys
sudo nano /home/grader/.ssh/authorized_keys
 (delete everyting before 'ssh -rsa' so that only the key remains)
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys
```
3. Give root ownership to grader and restart
```bash
sudo chown -R grader:grader /home/grader/.ssh
sudo service ssh restart
```
4. Check that you can login with grader account
```bash
ssh -i ~/.ssh/ LightsailDefault.pem  grader@[YOUR.PUBLIC.IP.ADDRESS]
```
#### ***Step 4: Configure firewall, ports, and other permissions***
1. Login as grader
```bash
ssh -v -i ~/.ssh/ LightsailDefault.pem  grader@[YOUR.PUBLIC.IP.ADDRESS]
```
2. Turn off password authentication
```bash
sudo nano /etc/ssh/sshd_config
```
3. Find PasswordAuthentication and set as No
4. Configure firewall to allow certain port numbers and port types
```bash
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/upd
sudo ufw enable
```
5. Go to your instance web page and find the Networking tab. Add to firewall: Custom TCP 2200
6. Change the SSH port from 22 to 2200
```bash
sudo nano /etc/ssh/sshd_config
```
- Find the port line and add Port 2200 and remove Port 22
- try to login with grader using port 2200 with
```bash
ssh -v -i ~/.ssh/ LightsailDefault.pem  grader@[YOUR.PUBLIC.IP.ADDRESS]-p 2200

```
7. Disable ssh login for root user
```bash
sudo nano /etc/ssh/sshd_config
```
- Find the PermitRootLogin line and set it to no.
- Restart apache service
```bash
sudo service ssh restart 
```
8. add DenyUsers ubuntu to the sshd_config file

#### ***Step 5: Install and configure Apache to serve a Python mod_wsgi application***
1. Install Apache web server:
```bash
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
sudo service apache2 start
```

2. Install Git:
```bash
sudo apt-get install git
git config --global user.name [YOUR GIT USERNAME]
git config --global user.email [YOUR GIT EMAIL]
sudo cd /var/www
sudo git clone [CLONE URL OF YOUR CATALOG PROJECT ON GITHUB] catalog
```
3.Rename the directory
```bash
sudo mv Item-Catalog  ItemCatalog
 ```

4. Configure wsgi:
-Make sure your folder (called ItemCatalog) containing all the code is inside a directory with this structure: /var/www/ ItemCatalog /
- Configure catalog.wsgi
```bash
sudo cd /var/www/catalog
sudo nano catalog.wsgi
```
-Add this to catalog.wsgi
```bash
#!/usr/bin/python3
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/ItemCatalog/")

from project import app as application
application.secret_key = 'super_secret_key'

```
5.Install all the libraries you need for your python application
```bash
sudo cd /var/www/ItemCatalog
sudo mv project.py __init__.py
sudo apt-get install python-pip
sudo pip install virtualenv
sudo virtualenv venv
source venv/bin/activate
sudo pip install Flask -t /var/www/ItemCatalog/venv/lib/python2.7/site-packages
sudo pip install [other python packages you might need] -t /var/www/ItemCatalog/venv/lib/python2.7/site-packages
(Suggestions: flask_oauth, httplib2, oauth2client, sqlalchemy, psycopg2, sqlalchemy_utils)
sudo nano /etc/apache2/sites-available/project.conf
```
- Change 
```bash
from project import app as application
```
To
```bash
from __init__ import app as application
```
In catalog.wsgi

- In /etc/apache2/sites-available/project.conf, add:
```bash
<VirtualHost *:80>
ServerName 18.184.151.245
ServerAlias ec2-18-184-151-245.eu-central-1.compute.amazonaws.com
ServerAdmin admin@18.184.151.245
WSGIDaemonProcess ItemCatalog python-path=/var/www/ItemCatalog:/var/www/ItemCatalog/venv/lib/python2.7/site-packages
WSGIProcessGroup ItemCatalog
WSGIScriptAlias / /var/www/ItemCatalog/catalog.wsgi
<Directory /var/www/ItemCatalog/>
Order allow,deny
Allow from all
</Directory>
ErrorLog ${APACHE_LOG_DIR}/error.log
LogLevel warn
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Restart apache service
```bash
sudo service apache2 reload
sudo a2ensite project
```
#### ***Step 6: Install and configure the database***
1. Install and Config PostgreSQL

```bash
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog with OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
Exit
```
2. Modify your python files for the newly configured postgresql database
```bash
cd /var/www/ItemCatalog
sudo nano __init__.py
```
2a.  Find the line "engine = create_engine ..." and change to
```bash
engine = create_engine('postgresql://catalog:password@localhost/catalog')
```
2b. Repeat Step 2 and 2a if you have a database set up file and/or a database populate file.
3.Restart apache service with 
```bash
sudo service apache2 reload
```
#### ***Step 7: Update OAuth authorized JavaScript origins***
1. To get the Google+ authorization working:
- Go to the project on the Developer Console: https://console.developers.google.com/project 
- Navigate to APIs & auth > Credentials > Edit Settings
- add your host name and public IP-address to your Authorized JavaScript origins and your host name + oauth2callback to Authorized redirect URIs, e.g. http://ec2-18-184-151-245.eu-central-1.compute.amazonaws.com/gconnect 

#### ***Step 8: Run the application***
1. Run your database set up file and/or datase populate file
```bash
sudo python3 /var/www/ItemCatalog/database_setup.py
sudo python3 /var/www/ItemCatalog/ item_catlog_init.py
```
2. Run your Flask application
```bash
sudo python3 /var/www/ItemCatalog/ __init__.py
```
3. Go to http://ec2-18-184-151-245.eu-central-1.compute.amazonaws.com to see if things worked.

Thanks to [Jessica Hsu](https://github.com/jessica-hsu/Full-stack-Nanodegree/tree/master/project7) for her helpful README

