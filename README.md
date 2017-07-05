# Linux Server Configuration
This project was completed as part of the Udacity FSND Program.

## Objective
In this project, we take a baseline installation of a Linux distribution on a virtual machine and prepare it to host web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.
It deploys the previously completed [Item Catalog App](https://github.com/megfh/Item-Catalog).


## Instructions for SSH access to the instance
1. Download Private Key below
2. Move the private key file into the folder `~/.ssh` (where ~ is your environment's home directory). So if you downloaded the file to the Downloads folder, just execute the following command in your terminal.
    ```mv ~/Downloads/project7 ~/.ssh/```
3. Open your terminal and type in
    ```chmod 600 ~/.ssh/project7```
4. In your terminal, type in
    ```ssh -i ~/.ssh/project7 ubuntu@13.59.81.129```
5. Development Environment Information

    Public IP Address

    13.59.81.129

    Private Key (provided in submission notes)

## Update and Upgrade all currently installed packages
  ```
  sudo apt-get update
  sudo apt-get upgrade
  ```

## Create a new user and give sudo permission
```sudo adduser grader --disabled-password```
Create a new file under the suoders directory:
```sudo nano /etc/sudoers.d/grader```.
Fill that newly created file with the following line of text: "grader ALL=(ALL) NOPASSWD:ALL", then save it. The grader user now has sudo permission

## Configure the key-based authentication for grader user
On your local machine, use `ssh-keygen` to generate an encryption key and save it as *~/.ssh/graderKey*
```
sudo su - grader
mkdir .ssh
chmod 700 .ssh
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
nano .ssh/authorized_keys
```
Paste the public key for your key pair you created into the file
Now to login remotely as grader:
```ssh -i ~/.ssh/graderKey grader@13.59.81.129```

[Source: Managing User Accounts on Your Linux Instance](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html)

## Configure the local timezone to UTC
Open time configuration dialog and set it to UTC
```sudo dpkg-reconfigure tzdata```

## Enforce key-based authentication
`sudo nano /etc/ssh/sshd_config` Find the PasswordAuthentication line and edit it to no. Then
```sudo service ssh restart```

## Change the SSH port from 22 to 2200
`sudo nano /etc/ssh/sshd_config` Find the Port line and edit it to 2200.
```sudo service ssh restart```

*note: port 2200 must also be allowed under the Firewall settings in the AWS Lightsail Console
Now you are able to log into the remote VM through ssh with the following command:
```ssh -i ~/.ssh/graderKey -p 2200 grader@13.59.81.129```

## Disable ssh login for root user
```sudo nano /etc/ssh/sshd_config```
Find the PermitRootLogin line and edit it to no. Then
```sudo service ssh restart```

## Configure the Uncomplicated Firewall (UFW)
Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
```
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
```

## Automatically manage package updates
Install unattended-upgrades if not already installed:
```sudo apt-get install unattended-upgrades```
To enable it, do:
```sudo dpkg-reconfigure --priority=low unattended-upgrades```

[Source: DigitalOcean](https://www.digitalocean.com/community/questions/auto-upgrade-packages-on-ubuntu)


## Install and configure Apache to serve a Python mod_wsgi application
Install Apache web server:
```sudo apt-get install apache2```
Install mod_wsgi for serving Python apps from Apache and the helper package python-setuptools:
```sudo apt-get install python-setuptools libapache2-mod-wsgi```
Restart the Apache server for mod_wsgi to load:
```sudo service apache2 restart```
Extend Python with additional packages that enable Apache to serve Flask applications:
```sudo apt-get install libapache2-mod-wsgi python-dev```
Enable mod_wsgi (if not already enabled):
```sudo a2enmod wsgi```

## Install Git
```sudo apt-get install git```
Configure your username:
```git config --global user.name <username>```
Configure your email:
```git config --global user.email <email>```

## Clone the Catalog app from github
```
cd /var/www
sudo mkdir catalog
sudo chown -R grader:grader catalog
cd catalog
git clone https://github.com/megfh/Item-Catalog catalog
```
*Note: Some tweaks were needed to deploy the item catalog app, so I made a deployment branch which slightly differs from the master*

Move inside the repository
```cd /var/www/catalog/catalog```
and change branch with:
```git checkout deployment```


## Setting up Flask & the virtual environment
Rename application.py to __init__.py
```
sudo mv application.py __init__.py
sudo apt-get install python-pip
sudo apt-get install python-virtualenv
cd /var/www/catalog
sudo virtualenv venv
source venv/bin/activate
sudo chmod -R 777 venv
pip install Flask
```
Install all other project dependencies:
```pip install sqlalchemy oauth2client httplib2 requests psycopg2```

## Configure and enable a new virtual host
Create a virtual host conifg file:
```sudo nano /etc/apache2/sites-available/catalog.conf```
Paste in the following:
```
<VirtualHost *:80>
    ServerName 13.59.81.129
    ServerAlias ec2-13-59-81-129.us-east-2.compute.amazonaws.com
    ServerAdmin admin@13.59.81.129
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
Enable the new virtual host:
```sudo a2ensite catalog```

Create wsgi file:
`cd /var/www/catalog` and
`sudo nano catalog.wsgi`
then paste in the following:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'Your secret key'
```

[Source: Flask Docs](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/)

[Source: DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

## Install and configure PostgreSQL
```
sudo apt-get update
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
```
Double check that no remote connections are allowed
```sudo nano /etc/postgresql/9.5/main/pg_hba.conf```
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```
Configure the new catalog user and database
```
sudo su - postgres
psql
```
```
# CREATE USER catalog WITH PASSWORD 'somepassword';
# ALTER USER catalog CREATEDB;
# CREATE DATABASE catalog WITH OWNER catalog;
# \c catalog
# REVOKE ALL ON SCHEMA public FROM public;
# GRANT ALL ON SCHEMA public TO catalog;
```
Exit out of PostgreSQl and the postgres user:
`# \q` then `exit`

Inside the Flask application (and database_setup.py), the database connection is now performed with:
```
engine = create_engine('postgresql://catalog:somepassword@localhost/catalog')
```

Create postgreSQL database schema:
```
python database_setup.py
```

[Source: DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

## Get O-Auth Login Working
Open http://www.hcidata.info/host2ip.cgi and receive the Host name for your public IP-address, e.g. for 13.59.81.129, its ec2-13-59-81-129.us-east-2.compute.amazonaws.com
To get the Google+ authorization working:
Go to the project on the Developer Console: https://console.developers.google.com/project
Navigate to APIs & auth > Credentials > Edit Settings
add your host name and public IP-address to your Authorized JavaScript origins and your host name + oauth2callback to Authorized redirect URIs, e.g. http://ec2-13-59-81-129.us-east-2.compute.amazonaws.com/oauth2callback
Make sure to update the client_secrets.json file in your catalog directory accordingly

## Restart Apache to launch the app
```sudo service apache2 restart```


## Credits:
All the aforementioned sources as well as several different posts on the Udacity Discussion Forums were very helpful in completing this project.


