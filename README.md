# FSND-P5-Linux-Server-Configuration

## IP, SSH port of my Server

  - IP: 52.36.32.89
      - or visit http://ec2-52-36-32-89.us-west-2.compute.amazonaws.com/
  - SSH port: 2200
      - login via `$ ssh -i ~/.ssh/grader.rsa grader@52.36.32.89 -p 2200`

### 1 & 2 - Create Development Environment: Launch Virtual Machine and SSH into the server

1. Create new development environment.
2. Download private keys and write down your public IP address.
3. Move the private key file into the folder ~/.ssh:
  `$ mv ~/Downloads/udacity_key.rsa ~/.ssh/`
4. Set file rights (only owner can write and read.):
  `$ chmod 600 ~/.ssh/udacity_key.rsa`
5. SSH into the instance:
  `$ ssh -i ~/.ssh/udacity_key.rsa root@52.36.32.89`

### 3 & 4 - User Management: Create new user `grader` and give permission to sudo

1. Create a new user:
  `$ adduser grader`
2. Give new user the permission to sudo
  1. Open the sudo configuration:
    `$ visudo`
  2. Add the following line below `root ALL...`:
    `grader ALL=(ALL:ALL) ALL`

### 5 - Update and upgrade all currently installed packages

1. Update the list of available packages and versions:
  `$ sudo apt-get update`
2. Install newer vesions of packages you have:
  `$ sudo sudo apt-get upgrade`

### 6 - Change the SSH port from 22 to 2200 and configure SSH access

1. Change ssh config file:
  1. Open the config file:
    `$ vim /etc/ssh/sshd_config`
  2. Change to Port 2200.
  3. Change `PermitRootLogin` from `without-password` to `no`.
  4. Temporarily change `PasswordAuthentication` from `no` to `yes`.
  5. Append `UseDNS no`.
  6. Append `AllowUsers grader`.
2. Restart SSH Service:
  `$ service ssh restart`
  1. Generate a SSH key pair on the local machine:
    `$ ssh-keygen`
  2. Copy the public id to the server:
    `$ ssh-copy-id grader@52.36.32.89 -p 2200`
  3. Login with the new user:
    `$ ssh -v grader@52.36.32.89 -p2200`
  4. Open SSHD config:
    `$ sudo vim /etc/ssh/sshd_config`
  5. Change `PasswordAuthentication` back from `yes` to `no`.

### 7 - Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

1. Check firewall status
  `$ sudo ufw status`
2. Allow incoming TCP packets on port 2200 (SSH):
  `$ sudo ufw allow 2200/tcp`
3. Allow incoming TCP packets on port 80 (HTTP):
  `$ sudo ufw allow 80/tcp`
4. Allow incoming UDP packets on port 123 (NTP):
  `$ sudo ufw allow 123/udp`
5. Enable firewall
  `$ sudo ufw enable`

### 8 - Configure the local timezone to UTC

1. Open the timezone selection dialog:
  `$ sudo dpkg-reconfigure tzdata`
2. Then chose 'None of the above', then UTC.

### 9 - Install and configure Apache to serve a Python mod_wsgi application

1. Install Apache web server:
  `$ sudo apt-get install apache2`
2. Open a browser and open your public ip address, e.g. http://52.25.0.41/ - It should  say 'It works!' on the top of the page.
3. Install **mod_wsgi** for serving Python apps from Apache and the helper package **python-setuptools**:
  `$ sudo apt-get install python-setuptools libapache2-mod-wsgi`
4. Restart the Apache server for mod_wsgi to load:
  `$ sudo service apache2 restart`
5. Get rid of the message "Could not reliably determine the servers's fully qualified domain name" after restart
  1. Create an empty Apache config file with the hostname:
    `$ echo "ServerName HOSTNAME" | sudo tee /etc/apache2/conf-available/fqdn.conf`
  2. Enable the new config file:
    `$ sudo a2enconf fqdn`

### 11 - Install git, clone and setup your Catalog App project

#### 11.1 - Install and configure git

1. Install Git:
  `$ sudo apt-get install git`
2. Set your name, e.g. for the commits:
  `$ git config --global user.name "liuderchi"`
3. Set up your email address to connect your commits to your account:
  `$ git config --global user.email "liuderchi@gmail.com"`

#### 11.2 - Setup for deploying a Flask Application on Ubuntu VPS

1. Extend Python with additional packages that enable Apache to serve Flask applications:
  `$ sudo apt-get install libapache2-mod-wsgi python-dev`
2. Enable mod_wsgi (if not already enabled):
  `$ sudo a2enmod wsgi`
3. Create a Flask app:
  1. Move to the www directory:
    `$ cd /var/www`
  2. Setup a directory for the app, e.g. catalog:
    1. `$ sudo mkdir catalog`
    2. `$ cd catalog` and `$ sudo mkdir catalog`
    3. Create the file that will contain the flask application logic:
      - `$ sudo nano __init__.py`
      - __NOTE__ file name must be `__init__.py`
    4. Paste in the following code:
    ```python
      from flask import Flask
      app = Flask(__name__)
      @app.route("/")
      def hello():
        return "Message From Flask!"
      if __name__ == "__main__":
        app.run()  # must in the main block
    ```
4. Install Required Packages
  1. Install pip installer:
    `$ sudo apt-get install python-pip`
  2. Install required packages
    `shell
    $ apt-get -qqy install postgresql python-psycopg2
    $ apt-get -qqy install python-sqlalchemy
    $ pip install Flask==0.9
    $ pip install oauth2client==1.5.2
    $ pip install requests
    $ pip install httplib2
    `

5. Configure and Enable a New Virtual Host#
  1. Create a virtual host config file
    `$ sudo nano /etc/apache2/sites-available/catalog.conf`
  2. Paste in the following lines of code and change names and addresses regarding your application:
  ```
    <VirtualHost *:80>
        ServerName 52.36.32.89
        ServerAdmin admin@52.36.32.89
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
  3. Enable the virtual host:
    `$ sudo a2ensite catalog`
6. Create the .wsgi File and Restart Apache
  1. Create wsgi file:
    `$ cd /var/www/catalog` and `$ sudo vim catalog.wsgi`
  2. Paste in the following lines of code:
  ```
    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/catalog/")

    from catalog import app as application
  ```
  7. Restart Apache:
    `$ sudo service apache2 restart`

#### 11.3 - Clone GitHub repository and make it web inaccessible
1. Clone project 3 solution repository on GitHub:
  `$ git clone https://github.com/liuderchi/fsnd_p3_travel_catalog.git`
2. Move all content of created FSND-P3_Music-Catalog-Web-App directory to `/var/www/catalog/catalog/`-directory and delete the leftover empty directory.
3. Make the GitHub repository inaccessible:
  1. Create and open .htaccess file:
    `$ cd /var/www/catalog/` and `$ sudo vim .htaccess`
  2. Paste in the following:
    `RedirectMatch 404 /\.git`

### 10 - Install and configure PostgreSQL

1. Install PostgreSQL:
  `$ sudo apt-get install postgresql postgresql-contrib`
2. Check that no remote connections are allowed (default):
  `$ sudo vim /etc/postgresql/9.3/main/pg_hba.conf`
3. Open the database setup file:
  `$ sudo vim database_setup.py`
4. Change the line starting with "engine" to (fill in a password):
  `python
  engine = create_engine('postgresql://catalog:mypassword@localhost/catalog')
  `
5. Change the same line in application.py respectively
6. Rename application.py:
  `$ mv application.py __init__.py`
7. Create needed linux user for psql:
  `$ sudo adduser catalog` (choose a password)
8. Change to default user postgres:
  `$ sudo -u postgres -i`
9. Connect to the system:
  `$ psql`
10. Add postgre user with password:
  1. Create user with LOGIN role and set a password:
    `# CREATE USER catalog WITH PASSWORD 'mypassword';` (# stands for the command prompt in psql)
  2. Allow the user to create database tables:
    `# ALTER USER catalog CREATEDB;`
  3. \*List current roles and their attributes:
    `# \du`
11. Create database:
  `# CREATE DATABASE catalog WITH OWNER catalog;`
12. Connect to the database catalog
  `# \c catalog`
13. Revoke all rights:
  `# REVOKE ALL ON SCHEMA public FROM public;`
14. Grant only access to the catalog role:
  `# GRANT ALL ON SCHEMA public TO catalog;`
15. Exit out of PostgreSQl and the postgres user:
  `# \q`, then `$ exit`
16. Create postgreSQL database schema:
  $ python database_setup.py

#### 11.5 - Run application
1. Restart Apache:
  `$ sudo service apache2 restart`
2. Open a browser and put in your public ip-address as url, e.g. 52.25.0.41 - if everything works, the application should come up
3. If getting an internal server error, check the Apache error files:
  1. View the last 20 lines in the error log:
    `$ sudo tail -20 /var/log/apache2/error.log`
  2. If a file like 'g_client_secrets.json' couldn't been found
    - add `os.chdir()` before `open()`

#### 11.6 - Get OAuth-Logins Working

1. Open http://www.hcidata.info/host2ip.cgi and receive the Host name for your public IP-address, e.g. for 52.25.0.41, its ec2-52-25-0-41.us-west-2.compute.amazonaws.com
2. Open the Apache configuration files for the web app:
  `$ sudo vim /etc/apache2/sites-available/catalog.conf`
3. Paste in the following line below ServerAdmin:
  `ServerAlias HOSTNAME`, e.g. ec2-52-25-0-41.us-west-2.compute.amazonaws.com
4. Enable the virtual host:
  `$ sudo a2ensite catalog`
5. To get the Google+ authorization working:
  1. Go to the project on the Developer Console: https://console.developers.google.com/project
  2. Navigate to APIs & auth > Credentials > Edit Settings
  3. add your host name and public IP-address to your Authorized JavaScript origins
    - e.g. http://52.36.32.89
  4. add your host name and public IP-address + oauth2callback to Authorized redirect URIs
    - e.g. http://ec2-52-36-32-89.us-west-2.compute.amazonaws.com/oauth2callback
6. download latest credentials json file and update content of my old `client_secret.json` in catalog directory


#### List of 3rd Party resource, and Special Thanks

Thanks to [stueken](https://github.com/stueken), preparing so much detailed walkthrough of project 5.

most of 3rd Party resource is come from his [walkthrough](https://github.com/stueken/FSND-P5_Linux-Server-Configuration)
