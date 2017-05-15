# linux-server-configuration

For the last project of Udacity's Full Stack Web Developer nanodegree, we needed to configure an Amazon lightsail linux (Ubuntu) instance to be a web server, and then deploy one of our previous Python projects (Item Catalog) to it.

Below are the steps that were taken to do so in my particular case, which would differ slightly for someone else trying to do the same thing (IP address, for example, would be different).

## Server Details

IP address: 34.205.135.141

SSH port (after below changes): 2200

NOTE: use URL to visit site
URL: ec2-34-205-135-141.compute-1.amazonaws.com

### Connecting to Amazon lightsail instance with key

Download the default key in lightsail. Im my case I named the file udacity.pem and put it in the ~/.ssh directory

Modify permissions to 400 on the file
```
chmod 400 ~/.ssh/udacity.pem
```

Connect
```
ssh -i ~/.ssh/udacity.pem ubuntu@34.205.135.141 -p 2200
```

## Configuration changes

### Update all currently installed packages

Get available updates:
```
sudo apt-get update
```
Apply all updates
```
sudo apt-get upgrade
```

### Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
! Do this before changing ssh port to avoid locking yourself out !

```
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/tcp
sudo ufw enable
```

### Change the SSH port from 22 to 2200

Promote to root and edit /etc/ssh/sshd_config
```
sudo nano /etc/ssh/sshd_config
```
Remove comment from line listing port (22) and change to 2200.

Save and exit.

Restart ssh service
```
sudo service sshd restart
```
Check to make sure port was reconfigured (should be listening on port 2200)
```
sudo service ssh status
```

### Give grader access

Create a new user account named grader and set a password (for the time being) when prompted.
```
sudo adduser grader
```
Give grader the permission to sudo.
```
sudo usermod -aG sudo grader
```
Setup SSH login for grader. Note that on lightsail the ubuntu user's authorized keys must be copied instead of root, otherwise ssh login will fail.
```
su - grader
mkdir .ssh
chmod 700 .ssh
sudo cp /home/ubuntu/.ssh/authorized_keys .ssh/authorized_keys
sudo chown grader:grader .ssh/authorized_keys 
chmod 644 .ssh/authorized_keys

```
Test grader login
```
ssh -i ~/.ssh/udacity.pem grader@34.205.135.141
```
### Prepare to deploy project

Configure the local timezone to UTC (follow the prompts). Note that when using lightsail the timezone is already set to UTC...
```
sudo dpkg-reconfigure tzdata
```
Install Apache and libapache2-mod-wsgi
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
```

Install PostgreSQL
```
sudo apt-get install postgresql postgresql-contrib

```
Do not allow remote connections. Remote connections are not allowed by default. This information can be found in:
```
/etc/postgresql/9.5/main/pg_hba.conf
```

Create a new database user named catalog that has limited permissions to catalog application database
```
sudo -u postgres createuser -P catalog
sudo -u postgres createdb -O catalog catalog
```
Install git (already installed by default, and was up to date due to earlier system updates)
```
sudo apt-get install git
```

Configure Apache to serve a Python (in my case Flask) mod_wsgi application ([reference](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps))
1) Create and configure new virtual host by creating /etc/apache2/sites-available/catalog.conf: 
```
<VirtualHost *:80>
        ServerName 34.205.135.141
        ServerAdmin admin@34.205.135.141
        ServerAlias ec2-34-205-135-141.compute-1.amazonaws.com
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
2) Create the .wsgi file and add:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super secret key'
```

3) Enable virtual host:
```
sudo a2ensite catalog
sudo service apache2 restart
```
4) Install Flask

```
sudo apt-get install python-pip
sudo pip install virtualenv
sudo virtualenv venv
source venv/bin/activate
sudo pip install Flask
```

5) Install other software needed by catalog project
```
sudo pip install sqlalchemy
sudo pip install oauth2client
sudo pip install requests
suod pip install psycopg2
```

### Deploy the Item Catalog project

1) Clone git repo to /var/www/catalog and rename folder/project to catalog

2) Rename main.py
```
sudo mv main.py __init__.py
```
3) Change client_secrets.json (and FB) path in init py file to an absolute path instead
of relative:
```
/var/www/catalog/catalog/client_secrets.json
/var/www/catalog/catalog/fb_client_secrets.json
```
4) Add app's IP to Facebook's allowed authorized redirect URIs.

5) Restart Apache
```
sudo service apache2 restart
```
6) Visit http://ec2-34-205-135-141.compute-1.amazonaws.com/

### Cannot log in as root remotely
Edit /etc/ssh/sshd_config - change PermitRootLogin to no:
```
PermitRootLogin no
```
### Enforce Key-based SSH authentication
Edit /etc/ssh/sshd_config - already configured, but should read:

```
PasswordAuthentication no
```
Restart ssh for above two changes to take hold
```
sudo service ssh restart
```
