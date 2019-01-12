#Deploying_Linux_Servers
##Description
>This project is about learning a baseline installation of a Linux server and prepare it to host my catalog project web applications. Then I will secure my server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it

##The IP address and SSH port so your server can be accessed by the reviewer
>IP address: 13.233.233.138
>SSH Port:2200

##The complete URL to your hosted web application
>http://13.233.233.138

##A summary of software you installed and configuration changes made
1. Get my linux server.
>I creat an acount on Amazon Lightsail and Start a new Ubuntu Linux server instance

2. Secure my server.
	1. Update all currently installed packages.
		>Run the following command:
			1. `$ sudo apt-get update`.
			2. `$ sudo apt-get upgrade`.
	2. Change the SSH port from 22 to 2200. Make sure to configure the Lightsail firewall to allow it.
		1. `$ sudo nano /etc/ssh/sshd_config`. Find the *Port* line and edit it to *2200*.
		2. `$ sudo service ssh restart`.
	3. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
		1. `$ sudo ufw allow 2200/tcp`.
		2. `$ sudo ufw allow 80/tcp`.
		3. `$ sudo ufw allow 123/udp`.
		4. `$ sudo ufw enable`.

3. Give grader access.
	1. Create a new user account named grader.`$ sudo adduser grader`.
	2. Give grader the permission to sudo.
		>Create a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL:ALL) ALL", then save it.
	3. Create an SSH key pair for grader using the ssh-keygen tool.
		1. Generate an encryption key **on your local machine** with: `$ ssh-keygen -f ~/.ssh/Private-key.rsa`.
		2. Log into the remote VM as *root* user through ssh and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
		3. Copy the content of the *udacity_key.pub* file from your local machine to the */home/grader/.ssh/authorized_keys* file you just created on the remote VM. Then change some permissions:
			1. `$ sudo chmod 700 /home/grader/.ssh`.
			2. `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
			3. Finally change the owner from *root* to *grader*: `$ sudo chown -R grader:grader /home/grader/.ssh`.
			4. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/Private-key.rsa grader@52.34.208.247`.

4. Prepare to deploy my project.
	1. Configure the local timezone to UTC.`$ sudo dpkg-reconfigure tzdata`
	2. Install and configure Apache to serve a Python mod_wsgi application.
		1. `$ sudo apt-get install apache2`.
		2. Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications. Install *mod_wsgi* with the following command:
			`$ sudo apt-get install libapache2-mod-wsgi python-dev`.
		3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
		4. `$ sudo service apache2 start`.
	3. Install and configure PostgreSQL
		1. Install some necessary Python packages for working with PostgreSQL: `$ sudo apt-get install libpq-dev python-dev`.
		2. Install PostgreSQL: `$ sudo apt-get install postgresql postgresql-contrib`.
		3. Postgres is automatically creating a new user during its installation, whose name is 'postgres'. That is a tusted user who can access the database software. So let's change the user with: `$ sudo su - postgres`, then connect to the database system with `$ psql`.
	4. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD 'sillypassword';`.
	5. Give *catalog* user the CREATEDB capability: `# ALTER USER catalog CREATEDB;`.
	6. Create the 'catalog' database owned by *catalog* user: `# CREATE DATABASE catalog WITH OWNER catalog;`.
	7. Connect to the database: `# \c catalog`.
	8. Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`.
	9. Lock down the permissions to only let *catalog* role create tables: `# GRANT ALL ON SCHEMA public TO catalog;`.
	10. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`.
	11. Inside the Flask application, the database connection is now performed with:
```python
engine = create_engine('postgresql://catalog:sillypassword@localhost/catalog')
```
	12. Setup the database with: `$ python /var/www/catalog/catalog/setup_database.py`.
	13. To prevent potential attacks from the outer world we double check that no remote connections to the database are allowed. Open the following file: `$ sudo nano /etc/postgresql/9.3/main/pg_hba.conf` and edit it, if necessary, to make it look like this:
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```
Source: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps).

	4. Install git.
		1. `$ sudo apt-get install git`.
		2. Configure your username: `$ git config --global user.name <username>`.
		3. Configure your email: `$ git config --global user.email <email>`.

7. Deploy the Item Catalog project.
	1. Clone and setup your Item Catalog project from the Github repository
		1. `$ cd /var/www`. Then: `$ sudo mkdir catalog_project_V3`.
	2. Change owner for the *catalog* folder: `$ sudo chown -R grader:grader catalog_project_V3`.
	3. Move inside that newly created folder: `$ cd /catalog_project_V3` and clone the catalog repository from Github: `$ git clone https://github.com/mouzah828/catalog_project_V3`.
	4. Make a *catalog.wsgi* file to serve the application over the *mod_wsgi*. That file should look like this:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog_project_V3/")

from application import app as application
```
	2. Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser
		1. cd /etc/apache2/sites-available nano catalog.conf
			> I Type the code below in the file and save it with ctrl+X then Y
```python
<VirtualHost *:80>
  ServerName 13.233.233.138
  ServerAdmin admin@13.233.233.138
  WSGIScriptAlias / /var/www/catalog_project_V3/catalog.wsgi
  <Directory /var/www/catalog_project_V3/>
      Order allow,deny
      Allow from all
  </Directory>
  Alias /static /var/www/catalog_project_V3/catalog/static
  <Directory /var/www/catalog_project_V3/static/>
      Order allow,deny
      Allow from all
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/error.log
  LogLevel warn
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
		2. Disable the default Apache site, enable your flask app, and then restart Apache for the changes to take effect.
			>Run these commands to do this:
				`$ sudo a2dissite 000-default.conf`
				`$ sudo a2ensite myApp .conf`
				`$ sudo service apache2 reload`
			> When I check my browser I see Internal server error, and I use this command to solve the error
				`$ sudo tail -100 /var/log/apache2/error.log`
				I try my best to solve this issue but I still get this error *OperationalError: (sqlite3.OperationalError) unable to open database file (Backg
round on this error at: http://sqlalche.me/e/e3q8)*
				I find all solution to change the permission to full access but I still get the same error

##A list of any third-party resources you made use of to complete this project

I use this website to get instraction of how to Deploying python Flask web app on Amazon Lightsail
https://hk.saowen.com/a/0a0048ca7141440d0553425e8df46b16cdf4c13f50df4c5888256393d34bb1b9
