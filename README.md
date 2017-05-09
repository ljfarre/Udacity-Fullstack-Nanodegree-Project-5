# Udacity Linux Server Configuration Project
The Udacity Linux Server Configuration project steps are presented  below.  
This is the fifth and final module for the Full Stack nanodegree.  
I skipped the fourth module intentionally and will go back to finish it after
completing this module.  I took this approach because the Item Catalog project
in the third module is tightly linked with the Linux Server Configuration project
in module five.  Finally, I am only hosting one app on the server for now, but my plan
is to get all my projects running on the server before long.  I will maintain a Lightsail
instance or two for at least the next year or so.

Notes for reviewer:
* Public IP: `52.4.73.151`
* SSH PORT: `2200`
* Catalog App URL using DNS domain: http://farrettaclassics.com
* Grader Passphrase for SSH login: password

## Table of Contents
1. Introduction
2. Setting up an Ubuntu server in Lightsail
3. Updating server software, securing the server with UFW firewall and setting the timezone
4. Creating and securing the grader account with SSH
5. Setting up and configuring the Apache HTTP server and the Python wsgi interface
6. Setting up PostgreSQL server
7. Installing the Catalog app and preparing it for deployment - including all required modules
9. Running and testing the Catalog application
10. Configuring mod_status for server monitoring
11. References

### Introduction
The sections below attempt to cover all the steps necessary to build a server
and launch the catalog application, from module three of the Full Stack nanodegree,
in the cloud. Here we use Lightsail from Amazon Web Services (AWS) as the platform
of choice.  Every effort has been made to satisfy the rubric laid out for the
project.  This is essentially a porting exercise where we port Lee's Classic Car
Catalog from a VirtualBox/Vagrant environment running a guest Ububtu instance,
Apache and a SQLite database to an AWS/Lightsail environment running Ubuntu Linux,
Apache and a PostgreSQL database.


### Setting up an Ubuntu server in Lightsail
In order to launch an application on the web, an application hosting server is a
basic requirement.  Here we use Amazon Lightsail to configure a fully functioning,
low end Linux server. We could further virtualize this environment with venv but
we have chosen not to do so here.  The steps required to deploy a Lightsail Ubuntu
server are as follows:
1. Create a user account on AWS and select Lightsail from under the Compute section of the services page.
2. Find the create instance button, click on it, select the OS only option and choose Ubuntu.
3. For starters just use the default SSH key pair, select the lowest end server option (5$/month) and
click the create button.
4. Once the server is created, you can connect to the server with SSH using the Lightsail console initially.
5. The default Ubuntu account has admin access and will allow you to sudo as the root user.
6. Use the Ubuntu account until you have created the grader account and given it sudo access.
7. At this point you can `sudo su root` with the default Ubuntu account but it is not recommended.

I took an additional step here of creating a static IP address and creating a DNS A record using
AWS's Route 53 service.  This way the IP does not change when I reboot the server, and I have a customized domain
for the application.  This was helpful when updating the Authorized Redirect URI for Google's Oauth2.  It
will not accept a public IP address for the redirect path in the credentials configuration.

You should have a fully functional Ubuntu server up an running at this point.

### Updating server software, securing the server with UFW firewall and setting the timezone
Here we update the server software and configure the UFW firewall to make sure our server is secure.
1. Update all currently installed packages:
  - Find updates: `sudo apt-get update`
  - Install updates: `sudo apt-get upgrade` Hit Y for yes and give yourself a break while it installs.
2. Configure the Uncomplicated Firewall (UFW) to only allow  incoming connections for SSH (port 2200),
HTTP (port 80), and NTP (port 123):
  - Check UFW status to make sure its inactive: `sudo ufw status`
  - Deny all incoming by default: `sudo ufw default deny incoming`
  - Allow outgoing by default: `sudo ufw default allow outgoing`
  - Allow SSH: `sudo ufw allow ssh`
  - Allow SSH on port 2200: `sudo ufw allow 2200/tcp`
  - Allow HTTP on port 80: `sudo ufw allow 80/tcp`
  - Allow NTP on port 123: `sudo ufw allow 123/udp`
  - Turn on firewall: `sudo ufw enable`
3. Make sure the timezone is set to UTC:
  - Ceck that the timezone is set to UTC: `sudo dpkg-reconfigure tzdata`
  - It should default to UTC.

A word of caution here is in order.  Even after you have configured the ufw firewall in Ubuntu,
you need to make certain that the AWS Lightsail firewall ports are open for this instance.  On the
dashboard for your Lightsail instance, click on the Networking tab, scroll down to the Firewall
settings and make sure that you add the firewall settings above there as well.  When you are sure
SSH is working on port 2200, you should remove port 22.

### Creating and securing the grader account with SSH and sudo access
There are several ways that this could have been done.  Here I tried to
follow the class instructions as closely as possible.
1. Create a new user named grader:
  - Enter: `sudo adduser grader`
  - Give grader a password of: `password`
  - Install finger: `apt-get install finger`
  - Check grader account with finger: `finger grader`
2. Give the grader the permission to sudo:
  - Create a sudoers.d file for grader: `sudo nano /etc/suoders.d/grader`
  - Inside the file add: `grader ALL=(ALL) NOPASSWD:ALL`
  - Save file: (nano: `ctrl+x`, `Y`, Enter)
  - This gives grader the same level of admin access as the Ubuntu account which is created
    when the cloud instance of Ubuntu is built.
3. Create SSH keys and copy the public key to the server manually:
  - On your local machine generate SSH key pair with: `ssh-keygen`
  - I added a passphrase for security: `password`  (This is obviously not secure, but keeping it simple here.)
  - Save the public and private SSH keys in in the default SSH directory: `/c/Users/Lee Farretta/.ssh`  (in my case.)
  - My public and private key files are: `grader-farretta-01.pub` and `grader-farretta-01`
  - Login into grader account using password set during user creation: `ssh grader@52.4.73.151 -p 2200`
  - Make .ssh directory: `mkdir .ssh`
  - Make file to store public key: `touch .ssh/authorized_keys`
  - On your local machine read and copy contents of the public key: `cat ~/.ssh/grader-farretta-01.pub` then
    copy the key to  the clipboard.
  - Login to the grader account and paste contents of clipboard into the authorized_keys file:
    `nano /home/grader/.ssh/authorized_keys` then paste contents(ctr+v).
  - Save file: (nano: `ctrl+x`, `Y`, Enter)
  - Set permissions for the directory and file: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
  - Change `PasswordAuthentication` from `yes` back to `no` in the sshd_config file:  `nano /etc/ssh/sshd_config`
  - Save file(nano: `ctrl+x`, `Y`, Enter)
  - Login with the corresponding private key: `ssh grader@52.4.73.151 -p 2200 -i ~/.ssh/grader-farretta-01`

The server now has a fully functioning admin account named grader that can only login remotely using SSH with a public/private key pair.

### Setting up and configuring the Apache HTTP server and the Python wsgi interface
Here we install the Apache web server and make sure the default Apache web page can be
accessed on port 80.  The Apache and wsgi configurations are then added using these steps:
1. Install Apache web server: `sudo apt-get install apache2`
  - Check port 80 with the public IP address given above: `52.4.73.151`.
2. Install mod_wsgi: `sudo apt-get install libapache2-mod-wsgi`
  - Configure Apache to handle requests using the WSGI module: `sudo nano /etc/apache2/sites-enabled/000-default.conf`
  - Add: `WSGIScriptAlias / /var/www/html/myapp.wsgi` before `</VirtualHost>` closing line
  - Save file: (nano: `ctrl+x`, `Y`, Enter)
  - Restart Apache: `sudo apache2ctl restart`
3. Install python-dev package: `sudo apt-get install python-dev`
4. Verify wsgi is enabled: `sudo a2enmod wsgi`

### Setting up PostgreSQL server
Next the PostgreSQL is installed and configured.  This is required for the port of
the catalog database from SQLite to PostgreSQL.
1. The following steps get the catalog database up and running:
  - `sudo apt-get install libpq-dev python-dev`
  - `sudo apt-get install postgresql postgresql-contrib`
  - `sudo su - postgres`
  - `psql`
  - `CREATE USER catalog WITH PASSWORD 'password';`
  - `ALTER USER catalog CREATEDB;`
  - `CREATE DATABASE catalog WITH OWNER catalog;`
  - `\c catalog`
  - `REVOKE ALL ON SCHEMA public FROM public;`
  - `GRANT ALL ON SCHEMA public TO catalog;`
  - `\q`
  - `exit`
2. After you have installed the catalog app below, make sure that you:
  - Change create engine line in your `__init__.py` and `database_setup.py` to:
    `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
  - Make sure no remote connections to the database are allowed: `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`
    Should produce output that looks like this:
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

###  Installing the Catalog app code and preparing it for deployment
Here we clone the catalog app from GitHub, install it on the Lightsail Ubuntu
instance and configure it for production.
1. Install git using: `sudo apt-get install git`
  - `cd /var/www`
  - `sudo mkdir catalog`
  - Change owner of the newly created catalog folder `sudo chown -R grader:grader catalog`
  - `cd /catalog`
2. Clone your project from github `git clone https://github.com/ljfarre/lees-aws-catalog-db.git catalog`
3. Create a catalog.wsgi file, then place it in the /var/www/catalog directory.
  - Run this: `sudo nano /var/www/catalog/catalog.wsgi`
  - Paste the code below and save the file (nano: `ctrl+x`, `Y`, Enter):
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'lees_secret_key'
```
4. Go to the /var/www/catalog/catalog directory and rename the project.py file:
  - `cd /var/www/catalog/catalog`
  - `mv project.py __init__.py`
5. Install application dependencies and APIs from requirements.txt:
  - `sudo pip install -r /var/www/catalog/catalog/requirements.txt`
  - It will install the following modules if they are not already there:
  ```
  bleach==1.4.2
  dicttoxml==1.6.6
  dominate==2.1.16
  Flask==0.10.1
  Flask-Bootstrap==3.3.5.7
  Flask-SQLAlchemy==2.1
  Flask-WTF==0.12
  html5lib==0.9999999
  httplib2==0.9.2
  itsdangerous==0.24
  Jinja2==2.8
  MarkupSafe==0.23
  oauth2client==1.5.2
  psycopg2==2.6.1
  pyasn1==0.1.9
  pyasn1-modules==0.0.8
  rsa==3.2.3
  six==1.10.0
  SQLAlchemy==1.0.9
  visitor==0.1.2
  Werkzeug==0.11.2
  wheel==0.24.0
  WTForms==2.0.2
  ```

6. Make sure the Apache server is configured so that that the Catalog app is accessible on port 80:
  - Run this: `sudo nano /etc/apache2/sites-available/catalog.conf`
  - Paste the code below and save the file (nano: `ctrl+x`, `Y`, Enter):
  ```
  <VirtualHost *:80>
  ServerName 52.4.73.151
  ServerAlias farrettaclassics.com
  ServerAdmin ljfarre@att.net
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
7. As stated above, tune the Python code to work with PostgreSQL instead of SQLite:
  - Change create engine line in your `__init__.py` and `database_setup.py` to:
  `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
  -Run to set up database initially: `python /var/www/catalog/catalog/database_setup.py`

8. Make sure that Oauth2 is configured to work with the new URIs:
  - Go to the project on the Developer Console: https://console.developers.google.com/project
  - Navigate to APIs & auth > Credentials > Edit Settings
  - Add host name and public IP-address to your Authorized JavaScript origins and your
    host name + oauth2callback to Authorized redirect URIs, e.g. http://farrettaclassics.com
    and `52.4.73.151`
  - Very Important:  Edit `__init__.py` and make sure that the path to the client_secrets.json
    file is fully elaborated, e.g., change all instances to /var/www/catalog/catalog/client_secrets.json.

### Running the catalog application
Here we reboot the server and run the application:
  - Either from the Litsail console or the command line reboot your server to make sure all dependencies
    are running properly and that server memory and cache are cleared.
  - Then restart apache: `sudo service apache2 restart`
  - The following URL should now take you to Lee's Classic Car Catalog: http://farrettaclassics.com

At this point, you should be able to see the content that is already there, though it is
limited.  You should be to login with your Google credentials and create your
own classic car dealership and car inventory.

### Configuring mod_status for application monitoring
I used the apache module mod_status to monitor the performance and availability of the application. By default, mod_status was already installed and enable on my version of apache, so the only changes I had to make were adding these lines inside my virtual host configuration:
1. Run this: `sudo nano /etc/apache2/sites-available/catalog.conf`
2. Paste this code and save the file (nano: `ctrl+x`, `Y`, Enter):
```
<Location /server-status>
SetHandler server-status
Order deny,allow
Deny from all
Allow from localhost
</Location>
```
3. Restart apache: `sudo service apache2 restart`
4. In order to view mod_status' reports install lynx terminal-based browser:
  - `sudo apt-get install lynx`
5. View the reports with the command:
  - `lynx http://localhost/server-status`

### References
This README file was created from class notes, the project rubric requirements, several good examples in
GitHub and the links below:
* [Changing the timezone](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28terminal.29)
* [Configuring Postgresql](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
* [Configuring a flask application with Apache2 and mod-wsgi](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* [Monitoring apache2 with mod_status](http://www.tecmint.com/monitor-apache-web-server-load-and-page-statistics/)
