# Udacity Linux Server Configuration

This is the final project for the Udacity Full Stack Developer Nanodegree. 

The objective is to configure an Amazon Lightsail server instance to host a catalog web application previously created using Flask.

**IP Address:** 52.64.161.176

**URL:** http://52.64.161.176.xip.io/

## Summary of Software Installed

- Ubuntu 18.04.2 LTS
- Postgres
- Apache
- WSGI
- PIP
- Virtualenv
- Flask
- Flask-Sqlalchemy
- Oauth2client
- Requests
- Flask-Migrate
- Flask-Script
- Psycopg2

## Configuration Changes
### Update Packages

- Update package source list

```
sudo apt-get update
```

- Update installed packages
```
sudo apt-get upgrade
```

### SSH Port Change

- Open /etc/ssh/sshd_config
```
sudo nano /etc/ssh/sshd_config
```
- Change the line `Port 22` to `Port 2200` and uncomment it
- In the Amazon Lightsail Console, remove the SSH port 22 under "Networking" and add 2200
- Restart the ssh service
```
sudo service ssh restart
```

### Disable Password Login

- Open /etc/ssh/sshd_config 
```
sudo nano /etc/ssh/sshd_config
```
- Set `PasswordAuthentication` to `no`

### Root Login
Remote root login is disabled by default on Amazon Lightsail servers

**Reference**: https://au.godaddy.com/help/changing-the-ssh-port-for-your-linux-server-7306

### Firewall

- Deny all incoming traffic
```
sudo ufw default deny incoming
```
- Configure outgoing traffic
```
sudo ufw default allow outgoing
```
- Allow port 2200
```
sudo ufw allow 2200/tcp
```
- Allow HTTP traffic
```
sudo ufw allow www
```
- Allow NTP traffic
```
sudo ufw allow ntp
```
- Enable UFW
```
sudo ufw enable
```

**Reference**: https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands

### Grader User Account

- Create a new account for grader
```
sudo adduser grader --disabled-password
```
- Create a copy of the sudoers.d file of the current user and name it grader
```
sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
```
- Access `/etc/sudoers.d/grader` and change the username from `ubuntu` to `grader`
```
sudo nano /etc/sudoers.d/grader
```
- Change the context to the `grader` account
```
sudo su - grader
```
- Create a .ssh directory in the `grader` home directory
```
mkdir .ssh
```
- Change the permissions of the `.ssh` directory to 700
```
chmod 700 .ssh
```
- Create the `authorized_keys` file in the `ssh` directory
```
touch .ssh/authorized_keys
```
- Change the permissions of the `authorized_keys` file to 600
```
chmod 600 .ssh/authorized_keys
```
- Create a new SSH key in the Amazon Lightsail console for `grader` and download it to your local computer
- Change the permissions of the downloaded key to 400
```
chmod 400 <filename>
```
- Extract the public key from the downloaded key pair
```
ssh-keygen -y -f filename.pem
```
- Append the public key to the `.ssh/authorized_keys` file
```
cat >> .ssh/authorized_keys
```
**References**:

- https://aws.amazon.com/premiumsupport/knowledge-center/new-user-accounts-linux-instance/
- https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ec2-key-pairs.html#retrieving-the-public-key

### Local Timezone
Default is already UTC by default, and it can be confirmed by using the `date` command

### App Deployment
#### Postgres Setup

- Install Postgres
```
sudo apt-get install postgresql postgresql-contrib
```
- Switch to postgres user
```
sudo -i -u postgres
```
- Create a database called `catalog`
```
createdb catalog
```
- Access postgres
```
psql
```
- Create new user called `catalog`
```sql
create user catalog with password 'password';
grant all privileges on database catalog to catalog;
```
**References**:

- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04
- https://tableplus.io/blog/2018/04/postgresql-how-to-grant-access-to-users.html
- https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps


#### Apache and WSGI Set Up

- Install Apache
```
sudo apt-get install apache2
```
- Install WSGI
```
sudo apt-get install libapache2-mod-wsgi python-dev
```
- Enable WSGI
```
sudo a2enmod wsgi 
```
- Navigate the directory where the app will be hosted
```
cd /var/www
```
- Clone the Github project for the app 
```
sudo git clone https://github.com/iamaleaf/book-catalog-project.git Catalog
```
- Add the client secret and id, as well as database username and password to the config files as they are not included in the github project
- Install pip
```
sudo apt-get install python-pip 
```
- Install virtualenv
```
sudo pip install virtualenv 
```
- Set up a virtual environment in the Catalog directory
```
sudo virtualenv venv
```
- Activate the virtual environment
```
source venv/bin/activate 
```
- Change the owner of the venv and site-packages folder in the virtual environment to the current user ubuntu to allow for the required packages to be installed within the virtual environment
```
sudo chown ubuntu:ubuntu /var/www/Catalog/venv/*
sudo chown ubuntu:ubuntu /var/www/Catalog/venv/lib/python2.7/site-packages
```
- Install the required packages for the app to run
```
pip install flask
pip install flask-sqlalchemy
pip install oauth2client
pip install requests
```
- Deactivate the virtual environment
```
deactivate
```
- Create a conf file for Apache
```
sudo nano /etc/apache2/sites-available/Catalog.conf
```
- Add the following lines of code to the file:
```
<VirtualHost *:80>
        ServerName 54.64.161.176
        ServerAdmin email@email.com
        WSGIScriptAlias / /var/www/Catalog/catalog.wsgi
        <Directory /var/www/Catalog>
            Order allow,deny
            Allow from all
        </Directory>
        Alias /static /var/www/Catalog/app/static
        <Directory /var/www/Catalog/app/static/>
            Order allow,deny
            Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Enable the virtual host
```
sudo a2ensite Catalog
sudo systemctl reload apache2
```
- Create the .wsgi File
```
sudo nano /var/www/Catalog/catalog.wsgi
```
- Add the following lines of code to the file:
```
#!/usr/bin/python

activate_this = '/var/www/Catalog/venv/bin/activate_this.py'
with open(activate_this) as f:
 exec(f.read(), dict(__file__=activate_this))


import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/Catalog/")

from app import app as application
application.secret_key = 'Add your secret key'
```

**References**:

- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps#setup
- https://medium.com/@satyavinay456/deploying-flask-application-using-wsgi-server-with-apache2-on-aws-cloud-ubuntu-machine-b7a15ca25cff

#### Database Migration

- Install Flask-Migrate, Flask-Script, and psycopg2
```
sudo pip install flask-migrate
sudo pip install flask_script
sudo apt-get build-dep python-psycopg2
sudo pip install psycopg2-binary
```
- Run the migration commands
```
sudo python manage.py db init
sudo python manage.py db migrate
sudo python manage.py db upgrade
```
- Seed the database
```
python db_seed.py
```
- Restart the Apache service
```
sudo systemctl reload apache2
```

**Reference**: https://realpython.com/flask-by-example-part-2-postgres-sqlalchemy-and-alembic/


