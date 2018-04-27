# Udacity Full Stack Nanodegree - Configuring a Linux Server to host a web app

# Server Details
URL: http://ec2-18-222-7-210.us-east-2.compute.amazonaws.com

IP: `18.222.7.210`
SSH port: `2200`

# 3rd Party Resources Used
- [Python]([https://www.python.org/)
- [Apache2](https://help.ubuntu.com/lts/serverguide/httpd.html)
- [mod_wsgi](https://modwsgi.readthedocs.io/en/develop/)
- [PostgeSQL](https://www.postgresql.org/)
- [Flask](http://flask.pocoo.org/)
- [SQLAlchemy](https://www.sqlalchemy.org/)
- [Pip](https://pypi.org/project/pip/)
- [OAuth2Client](https://oauth2client.readthedocs.io/en/latest/)
- [Requests](http://docs.python-requests.org/en/master/)
- [Git](https://git-scm.com/)
- [Facebook API](https://developers.facebook.com/)
- [Google API](https://developers.google.com/)
- [Amazon Lightsail](https://aws.amazon.com/lightsail/)
- [Ubuntu](https://www.ubuntu.com/)

# Server Configuration Settings

## Update installed packages
```
apt-get update
```
```
apt-get upgrade
```
## Create grader User
```
useradd -m -s /bin/bash grader
```
Grant grader sudo Access
```
visudo
```
Add line "grader ALL=(ALL) NOPASSWD:ALL"
Save and close the file.
```
adduser grader sudo
```

## Set-up SSH Keys for Remote Access
Download private key from "Default" key pair in Lightsail.
On local machine - place downloaded key file in desired directory.
Change file permissions for key file.
```
chmod 600 [key file path]
```
Connect to Lightsail server.
```
ssh -i [key file path] ubuntu@18.222.7.210 -p 2200
```
Disable root login.
```
vim /etc/ssh/sshd_config
```
Find the PermitRootLogin line and change it from 'prohibit-password' to 'no'
Restart SSH service
```
service ssh restart
```

Generate new key pair for grader login on local machine
```
ssh-keygen
```
Give the new key pair a name. Press [ENTER] when prompted for no password.

Back in Lightsail server...
Allow grader to log in remotely
```
mkdir /home/grader/.ssh
```
```
chown grader:grader /home/grader/.ssh
```
Create a new authorized_keys file in the grader/.ssh directory
```
touch /home/grader/.ssh/authorized_keys
```
```
vim /home/grader/.ssh/authorized_keys
```
Copy the contents of the .pub key file generated for the grader and paste them into the new authorized_keys file. Save and exit the file.

Change owner permissions to authorized_keys file and .ssh directory.
```
chown grader:grader /home/grader/.ssh/authorized_keys
```
```
chmod 644 /home/grader/.ssh/authorized_keys
```
```
chmod 700 /home/grader/.ssh
```

## Change SSH port from 22 to 2200
Edit sshd_config to change port to 2200
```
edit /etc/ssh/sshd_config
```
Change "Port 22" to "Port 2200"

Important: Before proceeding - in Lightsail, add new "Custom" TCP port range "2200" to Firewall under the Network settings.

## Configure Uncomplicated Firewall (UFW)
By default deny incoming connections.
```
ufw default deny incoming
```
Allow outgoing connections.
```
ufw default allow outgoing
```
Allow SSH on port 2200.
```
ufw allow 2200
```
Allow HTTP on port 80.
```
ufw allow www
```
Allow NTP on port 123.
```
ufw allow ntp
```
Restart the ssh service.
```
service ssh restart
```
## Configure the Local Timezone to UTC
```
timedatectl set-timezone UTC
```

## Install Apache and mod_wsgi
Install Apache2.
```
install apache2
```
Install mod_wsgi.
```
apt-get install libapache2-mod-wsgi
```

## Install and configure PostgreSQL
```
apt-get install postgresql postgresql-contrib
```
Configure PostgeSQL to not all remote connections.
```
vim /etc/postgresql/9.5/main/pg_hba.conf
```
By default remote connections aren't allowed. Verify that only 127.0.0.1 for IPv4 and ::1 for IPv6 are allowed.

Create a new database user named "catalog" that has limited permissions to the catalog database.
```
-u postgres createuser -P catalog
```
Enter a password for the user that will be used to access the database.
Create the "catalog" database.
```
-u postgres createdb -O catalog catalog
```

## Install Flask, SQLAlchemy, etc
These are packages that are need to run the catalog application.
```
apt-get install python-flask
apt-get install python-sqlalchemy
apt-get install python-pip
pip install oauth2client
pip install requests
pip install httplib2
pip install flask-sqlalchemy
```

## Install Git and Clone the Item Catalog application
```
apt-get install git
```
Create a new directory for the Item Catalog application.
```
mkdir /var/www/html/flaskapp
```
Using git clone the Item Catalog application to the new directory. Update any references to SQLLite database to the new postgres "catalog" database.
```
postgresql://catalog:[password]@localhost/catalog
```

## Configure Server For the Item Catalog Application
Create a catalog.wsgi file in /var/www/html/flaskapp.
Add the following line in catalog.wsgi.
```
#!/usr/bin/python
import sys
import logging

sys.path.insert(0,"/var/www/html/flaskapp/catalog")

logging.basicConfig(stream=sys.stderr)

from catalog import app as application
application.secret_key = 'super secret'
```

### Configure Apache2 to serve the app.
In /etc/apache2/sites-available create a file named catalogapp.conf.
Add the following lines in catalogapp.conf
```
<VirtualHost *>
  ServerName 18.222.7.210
  ServerAdmin grader@18.222.7.210
  WSGIDaemonProcess catalog
  WSGIScriptAlias / /var/www/html/flaskapp/catalog/catalog.wsgi
  <Directory /var/www/html/flaskapp/catalog>
    WSGIProcessGroup catalog
    Require All Granted
  </Directory>
  Alias /static /var/www/html/flaskapp/catalog/static
  <Directory /var/www/html/flaskapp/catalog/static>
    Require All Granted
  </Directory>
  Errorlog ${APACHE_LOG_DIR}/error.log
  LogLevel warn
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
### Change the default page that is displayed by the server
```
a2dissite 000-default.conf
```
```
a2ensite catalogapp.conf
```
```
service apache2 reload
```

## Initial Database Set-up
Run the following Python files to set-up the catalog database.
Make sure references to SQLLite have been change to PostgreSQL database.
```
python database_setup.py
```
```
python insertcategories.py
```

## Change Google OAuth Client Secrets File
Edit the "client_secret" Google login file to add http://ec2-18-222-7-210.us-east-2.compute.amazonaws.com to the redirect_uris and javascript_origins. This will allow the Google login to work correctly on the site.
Also, the url will need to be added in the Google Developer console settings for the app.

## Update Facebook App Settings
Using the Facebook Developer console add http://ec2-18-222-7-210.us-east-2.compute.amazonaws.com to the "Valid OAuth Redirect URIs" under the "Facebook Login Settings" for the app.

## Access Application
Open browser and navigate to http://ec2-18-222-7-210.us-east-2.compute.amazonaws.com you should see the main page of the Item Catalog application! :laughing::laughing::laughing:
