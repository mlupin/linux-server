# Linux Server Configuration

Public IP: 54.202.10.79
SSH Port: 2200
URL:  http://54.202.10.79

#### Installed Software:
- apache2
- libapache2-mod-wsgi
- PostgreSQL
- Flask
- git
- httplib2
- requests
- oauth2client
- python-psycopg2
- postgresql-contrib
- virtualenv

###### Create and connect.
1. Start a new Ubuntu Linux server instance on Amazon Lightsail.
2. Launch Amazon Lightsail terminal
	- Download the default key-pair and move to /.ssh folder.
	- Rename the key pair to 'sshkey' (optional) 
	- Open terminal and type in `chmod 600 ~/.ssh/sshkey.pem`
	- Use the command `ssh ubuntu@54.202.10.79 -i ~/.ssh/sshkey.pem`

###### Secure your server.
3. Update all currently installed packages.
	- Download package lists with `sudo apt-get update`
	- Fetch new versions of packages with `sudo apt-get upgrade`

4. Change the SSH port from 22 to 2200. Make sure to configure the Lightsail firewall to allow it.
	- Run `sudo nano /etc/ssh/sshd_config`
	- Change the port from 22 to 2200

5. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
	- Allow incoming connections
	```sudo ufw allow 2200/tcp
	sudo ufw allow 80/tcp
	sudo ufw allow 123/udp
	sudo ufw enable```
	- Check Uncomplicated Firewall (UFW) status `sudo ufw status`
	```Status: active

	To                         Action      From
	--                         ------      ----
	80/tcp                     ALLOW       Anywhere                  
	123/udp                    ALLOW       Anywhere                  
	2200/tcp                   ALLOW       Anywhere                  
	80/tcp (v6)                ALLOW       Anywhere (v6)             
	123/udp (v6)               ALLOW       Anywhere (v6)             
	2200/tcp (v6)              ALLOW       Anywhere (v6)```         

	- If connection is timed out or lost, use the command `ssh ubuntu@54.202.10.79 -p 2200 -i ~/.ssh/sshkey.pem` to ssh back in.

###### Give grader access.
6. Create a new user account named grader.
	- Run `sudo adduser grader` to create a new user named grader
	- Password: 'passw0rd'

7. Give grader the permission to sudo.
	- Create a new file in the sudoers directory with `sudo nano /etc/sudoers.d/grader`
	- Add the following text `grader ALL=(ALL:ALL) ALL`
	- Create a sudo file for grader: `sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader`
	- Open newly created grader file: `sudo nano /etc/sudoers.d/grader`
	- Edit the content as shown: 'grader ALL=(ALL) NOPASSWD:ALL'

8. Create an SSH key pair for grader using the ssh-keygen tool.
	- On the local machine, enter command `ssh-keygen` in Terminal
	- Enter file in which to save the key and enter passphrase: `ssh-keygen -y -f lightsail`
	- Back on the AWS Ubuntu server:
		* Switch to grader: `su - grader`
		* Create an .ssh folder: `mkdir .ssh`
		* Create an authorized_keys file: `touch .ssh/authorized_keys`
		* Copy rsa key into the file and save: `sudo nano .ssh/authorized_keys`
		* Run the following permissions on the folder and file
			`chmod 700 .ssh` `chmod 600 .ssh/authorized_keys`
		* Change 'PasswordAuthentification' to 'yes': `sudo nano /etc/ssh/sshd_config`
		* Logout, go back to ubuntu (root): `exit`
		* Restart the server: `sudo service ssh restart`
		* Login as 'grader': `ssh grader@54.202.10.79 -p 2200 -i ~/.ssh/lightsail`

###### Deploy a project.
9. Configure the local timezone to UTC.
	- Run `sudo dpkg-reconfigure tzdata` and then choose UTC
	- Scroll to the bottom and select `None of the above`; in the second list, select `UTC`
10. Install and configure Apache to serve a Python mod_wsgi application.
	- `sudo apt-get install apache2`
	- `sudo apt-get install libapache2-mod-wsgi`
11. Install and configure PostgreSQL:
	- Install python-dev package: `sudo apt-get install python-dev`
	- Enable mod_wsgi with `sudo a2enmod wsgi`
	- Start the web server with `sudo service apache2 start`

12. Install git.
	- `sudo apt-get install git`

###### Deploy the Item Catalog project.
13. Clone and setup your Item Catalog project from the Github repository.
	-Create flask app taken from [digitalocean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
		* `cd /var/www`
		* `sudo mkdir catalog`
		* `cd catalog`
		* `sudo mkdir catalog`
		* `cd catalog`
		* `sudo mkdir static templates`
		* `sudo nano __init__.py
			```from flask import Flask
			app = Flask(__name__)
			@app.route("/")
			def hello():
			    return "Hello, world (Testing!)"
			if __name__ == "__main__":
				app.run()```
	- Install flask
		* `sudo apt-get install python-pip`
		* `sudo pip install virtualenv`
		* `sudo virtualenv venv`
		* `sudo chmod -R 777 venv`
		* `source venv/bin/activate`
		* `pip install flask`
		* `pip install sqlalchemy`
		* `pip install Flask-SQLAlchemy`
		* `python __init__.py`
		* `deactivate`

	- Configure And Enable New Virtual Host
		* Create host config file `sudo nano /etc/apache2/sites-available/catalog.conf` paste the following:
			```<VirtualHost *:80>
			  ServerName 54.202.10.79
			  ServerAdmin admin@54.202.10.79
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
			    * save file(nano: `ctrl+x`, `Y`, Enter)
			    * Enable `sudo a2ensite catalog`

			* Create the wsgi file
			    * `cd /var/www/catalog`
			    * `sudo nano catalog.wsgi`

			    ```
			  #!/usr/bin/python
			  import sys
			  import logging
			  logging.basicConfig(stream=sys.stderr)
			  sys.path.insert(0,"/var/www/catalog/")

			  from catalog import app as application
			  application.secret_key = 'secret key'```

			* `sudo service apache2 restart`

		- Clone Github Repo and Made Revisions
			* `sudo git clone https://github.com/mlupin/fullstack-nanodegree-item-catalog`
			* `mv /var/www/catalog/catalog/fullstack-nanodegree-item-catalog/* /var/www/catalog/catalog/`
			* change create engine line in  __init__.py and database_setup.py to: `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
			* change client_secret.json file location in __init__.py

