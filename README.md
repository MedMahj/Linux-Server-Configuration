# Linux Server Configuration Project - Udacity FSWD Nanodegree

## Description 

Take a baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

* IP address: 52.47.194.202

* Accessible SSH port: 2200

* Application URL: http://ec2-52-47-194-202.eu-west-3.compute.amazonaws.com/

## Installed Packages

* Python 2
* PostgreSQL
* Apache2
* Apache2 WSGI Mod
* Git

## Steps To Complete This Configuration
### Create Ubuntu VM Instance
* Create an Amazon Lightsail instance using the OS Only + Ubuntu option.

* Download the default key pair file and log into the instance using SSH:

```ssh ubuntu@[IP-ADDRESS] -i [/PATH/TO/YOUR-KEY-PAIR-FILE]```

### Update packages
* Update all currently installed packages and linux distro:
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```
### Add Grader User

**On local machine:**

* Generate a key pair to be used by the **grader** user:

```
ssh-keygen
```

* Choose a location for the key pair file and save (e.g `~/.ssh/aws_grader`)

**On remote machine:**

* Add **grader** user:

```
sudo adduser grader
```

* Enter password and details for user

* Change to grader home directory and create an `.ssh` directory:

```
cd /home/grader
sudo mkdir .ssh
sudo nano .ssh/authorized_keys
```

* Copy and paste the contents of the `aws_grader.pub` file from the local machine and save

* Set permissions and owner/group on `.ssh` recursively:

```
sudo chown -R grader:grader .ssh
sudo chmod -R 700 .ssh
```
### Give **grader** sudo privileges

* Become the root user

```
sudo -i
```

* Create a new sudoer file in the `/etc/sudoers.d` directory:

```
nano /etc/sudoers.d/grader
```

* Copy and paste the following into the file and save:

```
grader ALL=(ALL) NOPASSWD:ALL
```

* Change permissions on file and exit sudo:

```
chmod 440 /etc/sudoers.d/grader
```

* Exit as root user:

```
exit
```
### Changing default SSH port and Forcing Key Based Authentication 

* Edit the `sshd_config` file:

```
sudo nano /etc/ssh/sshd_config
```

* Change port to 2200:

```
Port 2200
```

* Disable the root login:

```
PermitRootLogin no
```

* Enable authorized_keys to be used:

```
AuthorizedKeysFile      %h/.ssh/authorized_keys
```

* Disable password authentication:

```
PasswordAuthentication no
```

* Restart `ssh`:

```
sudo service ssh restart
```
### Configure Uncomplicated Firewall (UFW)

* Allow/deny access to ports:

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow www
sudo ufw allow ntp
sudo ufw allow 2200/tcp
sudo ufw enable
```

* Exit `ssh` session:

```
exit
```
### Allow connection to UFW ports for Lightsail instance

* Log in to your Lightsail dashboard

* Select the instance and click on Networking

* Under firewall, select edit rules

* Delete the SSH rule

* Select Add Another

* From the dropdown, select Custom TCP and enter 2200 for our new SSH port

* From the dropdown, select Custom TCP and enter 123 for our NTP port

* Save the changes

* SSH into your newly created and configured **grader** user on port 2200:

```
ssh grader@[IP-ADDRESS] -p 2200 -i [PATH/TO/GRADER-KEY-PAIR-FILE] -p 2200
```
* Enter passphrase for key : `grader`

### Configure Timezone

* Run the TimeZone setup tool and select your timezone (UTC):

```
sudo dpkg-reconfigure tzdata
```

### Install and configure Apache to serve a Python mod_wsgi application

* Install the apache2 web server and Python WSGI mod:

```
sudo apt-get install apache2 libapache2-mod-wsgi
```

* Disable default virtual servers:

```
sudo a2dissite *
```
### Install and configure PostgreSQL

* Install PostgreSQL:

```
sudo apt-get install postgresql postgresql-contrib
```
* Create a new database user named **catalog** that has limited permissions to catalog application database
  * Switch to postgres user:

  ```
  sudo su - postgres
  ```

  * Enter psql:

  ```
  psql
  ```
  * Create a new role named **catalog** and alter user to **catalog**:

  ```sql
  CREATE USER catalog WITH PASSWORD 'catalog'; 
  ALTER USER catalog CREATEDB; 
  ```
  * Create new database named **catalog**:

  ```sql
  CREATE DATABASE catalog WITH OWNER catalog; 
  ```
  * Connect to the database **catalog**:
  ``` 
  \c catalog 
  ``` 
  * Revoke all rights:
  ```sql
  REVOKE ALL ON SCHEMA public FROM public; 
  ``` 
  * Grant only access to the catalog role:
  ```sql
  GRANT ALL ON SCHEMA public TO catalog; 
  ``` 

  * Exit PSQL and log out of `postgres` user:

  ```
  \q
  exit
  ```
* Restart PostgreSQL
 ```
sudo /usr/sbin/service postgresql restart
 ```


### Install GIT

* Install the GIT package:

```
sudo apt-get install git
```
### Deploy the Item Catalog project
* Change directory to `/var/www`:

```
cd /var/www
```

* Create a new directory to contain the web application and `cd` into it:

```
sudo mkdir catalog
sudo chown -R grader:grader catalog
cd catalog
```

* Clone the GIT repo containing the web application files into a directory called **catalog**

```
sudo git clone https://github.com/MedMahj/Item_Catalog.git catalog
```
### Update Project Files and Setup database
* Change directory to `/var/www/catalog/catalog`
```
cd /var/www/catalog/catalog
```

* update application.py , model.py and database_setup.py to change from sqlite to postgresql and :
```
engine = create_engine('postgresql://catalog:catalog@localhost/catalog') 
```

* Rename application.py :
```
sudo mv application.py __init__.py
```
* Setup the database :
```
python /var/www/catalog/catalog/database_setup.py 
python /var/www/catalog/catalog/model.py 
```
### Configure Apache2 Server
* Installing Dependencies
```
sudo apt install python-pip
sudo pip install -r catalog/requirements.txt
```

* Create the `catalog.wsgi` file:

```
sudo nano var/www/catalog/catalog.wsgi
```

* Copy and paste this into the `catalog.wsgi` file and save it:

```python
import sys 
import logging 
logging.basicConfig(stream=sys.stderr) 
sys.path.insert(0, "/var/www/catalog/") 
from catalog import app as application 
application.secret_key = 'supersecretkey' 
```

* Create a file for the virtual host:

```
sudo nano /etc/apache2/sites-available/catalog.conf
```

* Copy and paste these contents into the file:

```
<VirtualHost *:80>  

    ServerName 52.47.194.202  
    ServerAlias ec2-52-47-194-202.eu-west-3.compute.amazonaws.com  
    ServerAdmin grader@52.47.194.202  
    WSGIDaemonProcess catalog  
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
* Enable the site:

```
sudo a2ensite catalog
```

* Restart the apache server:

```
sudo service apache2 restart
```
* Visit site at : 
  * http://ec2-52-47-194-202.eu-west-3.compute.amazonaws.com/ 
  * http://52.47.194.202/
## Authors

* **Mohamed BOUSETTA MAHJOUB** - *Initial work* - [MedMahj](https://github.com/MedMahj/)





