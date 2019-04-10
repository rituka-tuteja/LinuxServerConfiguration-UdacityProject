# Linux Server Configuration
This file contains information on how to take a baseline installation of a Linux distribution on a virtual machine and set it up to host the web and database applications.

- Ubuntu 16.04 LTS Linux distribution
- Amazon Lightsail virtual private server
- Restaurant Menu Web Application from Item Catalog Project
- PostgreSQl database server

## Setup
### Get your server
1. Create Lightsail account on https://lightsail.aws.amazon.com/ 
2. Create a Lightsail instance using Ubuntu instance image
3. In order to be able to SSH into the server, first download private key from 'Account' tab on Lightsail and move the key to .ssh folder on the local machine
4. From the Lightsail 'Networking' tab of the instance, add two new custom ports - 123/UDP and 2200/TCP

### SSH into the server
1. From the local terminal:
    rename the key:
    `$ mv ~/.ssh/LightsailDefaultPrivateKey-us-west-2.pem ~/.ssh/LightsailKey.pem`
    set the following permissions
    `$ chmod 600 ~/.ssh/LightsailKey.pem`
2. SSH into the server as ubuntu using the key
    `$ ssh -i ~/.ssh/LightsailKey.pem ubuntu@52.40.3.15`

### Create new account 'grader'
1. Logged in as ubuntu, create new user
    `$ sudo adduser grader`
2. Grant grader sudo permission
    `$ sudo nano /etc/sudoers.d/grader`
    Add `grader ALL=(ALL:ALL) ALL`
3. Update packages
    `$ sudo apt-get update`
    `$ sudo apt-get upgrade`
    `$ sudo apt-get install finger` (to install 'finger' package)
    `$ finger grader` (to get details about the user 'grader')

### Keygen for SSH key pair for grader
1. From the local terminal
    `$ ssh-keygen` (to get key pair, name it 'project_graderkey.rsa')
2. Logged in as ubuntu:
    `$ cd /home/grader`
    `$ mkdir .ssh`
    `$ touch .ssh/authorized_keys`
    `$ nano .ssh/authorized_keys` (copy the contents of project_graderkey.rsa public key and paste it in authorized_keys)
3. Permissions:
    `$ sudo chmod 700 /home/grader/.ssh`
    `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`
    `$ sudo chown -R grader:grader /home/grader/.ssh`
    `$ sudo service ssh restart`
    `$ ~.` (to disconnect)
4. Log in as grader from the local terminal using grader key
    `$ ssh -i ~/.ssh/project_graderkey.rsa grader@52.40.3.15`

### Enforce key based authentication
1. Edit and change the PasswordAuthentication settings to `no`
    `$ sudo nano /etc/ssh/sshd_config`
    `$ sudo service ssh restart`
2. Edit and change the Port line from `22` to `2200`
    `$ sudo nano /etc/ssh/sshd_config`
    `$ sudo service ssh restart`
    `$ ~.` (to disconnect)
3. Log in as grader using the port 2200
    `$ ssh -i ~/.ssh/project_graderkey.rsa -p 2200 grader@52.40.3.15`

### Disable root login
1. Edit and change the PermitRootLogin settings to `no`
    `$ sudo nano /etc/ssh/sshd_config`
    `$ sudo service ssh restart`

### Secure Server
1. Configure the firewall
    `$ sudo ufw allow 2200/tcp`
    `$ sudo ufw allow 80/tcp`
    `$ sudo ufw allow 123/udp`
    `$ sudo ufw enable`
    `$ sudo ufw status` (to check the status of the firewall)

## Prepare to Deploy the Project
### Install necessary packages 
1. Apache and GIT    
    `$ sudo apt-get install apache2`
    `$ sudo apt-get install libapache2-mod-wsgi python-dev`
    `$ sudo apt-get install git`
2. Enable mod_wsgi
    `$ sudo a2enmod wsgi`
    `$ sudo service apache2 start`
3. Install pip
    `$ sudo apt-get install python-pip`
4. Install flask and other packages
    `$ sudo apt-get install python-pip`  
    `$ sudo pip install Flask`  
    `$ sudo pip install Requests`  
    `$ sudo pip install httplib2`  
    `$ sudo pip install sqlalchemy`  
    `$ sudo pip install psycopg2`  
    `$ sudo pip install oauth2client`  
    `$ sudo pip install render_template`  
    `$ sudo pip install sqlalchemy_utils`  
    `$ sudo pip install redirect`

### Database
1. Install PostgreSQL
    `$ sudo apt-get install libpq-dev python-dev`  
    `$ sudo apt-get install postgresql postgresql-contrib`  
2. Configure PostgreSQL    
    `$ sudo su - postgres`  
    `$ psql`  
    `$ CREATE USER catalog WITH PASSWORD 'password';`
    `$ ALTER USER catalog CREATEDB;`
    `$ CREATE DATABASE catalog WITH OWNER catalog;`
3. Connect to database
    `$ \c catalog`
    `$ REVOKE ALL ON SCHEMA public FROM public;`
    `$ GRANT ALL ON SCHEMA public TO catalog;`
4. Quit the postgres command line:
    `$ \q` and then `$ exit`

### Virtual Machine
1. Install 
    `$ sudo pip install virtualenv`
2. Configure
    `$ sudo virtualenv venv`
    `$ source venv/bin/activate`
    `$ sudo chmod -R 777 venv`

### Configure virtual host
1. Create the file and add the following content:
    `$ sudo nano /etc/apache2/sites-available/catalog.conf`
    ```
        <VirtualHost *:80>
        ServerName 52.40.3.15
        ServerAlias ec2-52-40-3-15.us-west-2.compute.amazonaws.com
        ServerAdmin admin@52.40.3.15
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

### Deploy the project
1. Setup the directory structure
    `$ cd /var/www`
    `$ sudo mkdir catalog`
    `$ sudo chown -R grader:grader catalog`
    `$ cd catalog`
2. Clone Catalog Project
    `$ git clone https://github.com/rituka-tuteja/ItemCatalog-UdacityProject.git`
3. Create .wsgi file
    `$ sudo nano catalog.wsgi`
    ```
        #!/usr/bin/python
        import sys
        import logging
        logging.basicConfig(stream=sys.stderr)
        sys.path.insert(0, "/var/www/catalog/")

        from catalog import app as application
        application.secret_key = 'super_secret_key'
    ```
4. Rename the project.py to __init__.py
5. Edit __init__.py, database_setup.py and lotsofmenus.py files to change the database engine from
    `sqlite://catalog.db` to `postgresql://catalog:password@localhost/catalog`

### Google Authenticated Login
1. On https://console.developers.google.com, click Credentials
2. Click on Create credentials and select OAuth Clint ID
3. Choose 'Web Application'
4. Add `http://52.40.3.15`, `http://52.40.3.15.xip.io` to Authorized JavaScript Origins
5. Add `http://52.40.3.15.xip.io/login` and `http://52.40.3.15.xip.io/gdisconnect` to Authorised redirect URIs
6. Click create
7. Click Download Json to get the client_secrets.json file.
8. Copy the contents of client_secrets.json file to /var/www/catalog/catalog/client_secrets.json
9. Update the client Id in /var/www/catalog/catalog/templates/login.html
10. Change the client_secrets.json line to /var/www/catalog/catalog/client_secrets.json on line 22 and 53

### Launch the Application
1. Restart the Apache server
    `$ sudo service apache2 restart`
2. Run the Application
    `$ sudo python database_setup.py`
    `$ sudo python lotsofmenus.py`
    `$ sudo python __init__.py`

Visit the Web App from the browser at:
http://52.40.3.15:80
Login at
http://52.40.3.15.xip.io/login
Logout from:
http://52.40.3.15.xip.io/gdisconnect

### References:
https://www.youtube.com/watch?v=HcwK8IWc-a8
https://classroom.udacity.com/courses/ud299
http://httpd.apache.org/docs/current/configuring.html
https://www.hcidata.info/host2ip.cgi
https://help.ubuntu.com/community/Sudoers
https://github.com/boisalai/udacity-linux-server-configuration
https://github.com/bencam/linux-server-configuration
