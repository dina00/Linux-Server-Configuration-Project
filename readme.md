# Linux Server Configuration Project

## Intro
The objective is to take a Linux instance and modify it to host web applications, this includes installing updates, securing it from a number of attacks and installing/configuring web and database servers.
### Info About My Machine:
- IP: [52.14.112.201](52.14.112.201)
- DNS: [ec2-52-14-112-201.us-east-2.compute.amazonaws.com](ec2-52-14-112-201.us-east-2.compute.amazonaws.com)
- Port: 2200

## What You Need Beforehand
You need a Linux instance and Git bash to connect to it, also a Linux VM to act as your local machine.

## Instructions
#### 1. Update all packages
`
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
`
Enable automatic security updates:
`
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
`

#### 2. Change timezone to UTC and Fix language issues 
`
sudo timedatectl set-timezone UTC
sudo update-locale LANG=en_US.utf8 LANGUAGE=en_US.utf8 LC_ALL=en_US.utf8
`

#### 3. Create a new user grader and Give him `sudo` access
`
sudo adduser grader
sudo nano /etc/sudoers.d/grader 
`
Add the following text:  `grader ALL=(ALL) ALL`

#### 4. Setup SSH keys for grader
* On local machine run the command
`ssh-keygen`
Then choose the path for storing public and private keys
* On the linux instance (remote machine) switch to the user `grader` and perform the following:
```
sudo su - grader
mkdir .ssh
touch .ssh/authorized_keys 
sudo chmod 700 .ssh
sudo chmod 600 .ssh/authorized_keys 
nano .ssh/authorized_keys 
```
Copy the private key on your local machine. The goal is to have the local key on the remote machine and using the public key to access it.

#### 5. Change the SSH port from 22 to 2200 | Enforce key-based authentication | Disable login for root 
```
sudo nano /etc/ssh/sshd_config
```
Modify the following:
* Port from 22 to 2200.
* Set PasswordAuthentication to no.
* Set PermitRootLogin to no.
* Save the file and and restart ssh by  `sudo service ssh restart`.

#### 6. Configure the Uncomplicated Firewall (UFW)
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw allow 8000/tcp  `serve another app on the server`
sudo ufw enable
```

#### 7. Install Apache2 and mod-wsgi 
```
sudo apt-get install apache2 libapache2-mod-wsgi-py3 git
```

#### 8. Install and configure PostgreSQL
```
sudo apt-get install libpq-dev python3-dev
sudo apt-get install postgresql postgresql-contrib
sudo su - postgres
psql
```
Run the following `psql` commands:
```
CREATE USER catalog WITH PASSWORD 'password';
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit
```
**Note:** You should change database engine from Sqlite to Postgresql by replacing the create_engine function in all of the files in the project.
```
engine = create_engine('postgresql://catalog:password@localhost/catalog')
```

#### 9. Transfer your project from your local machine to the remote machine using SFTP :

1. On Git Bash run the command.
```sftp -P 2200 -i "key.pem" ubuntu@ip ``` 
2. Navigate to the directory containing your project folder.
```lcd \path\catalog```
3. Copy the folder to your instance.
```put -r catalog /home/grader/ ```

5. Create the WSGI file:
Create `catalog.wsgi` and add the following:
```python
#!/usr/bin/python3
import sys
sys.stdout = sys.stderr

# Add this if you'll create a virtual environment, So you need to activate it
# -------
activate_this = '/var/www/catalog/env/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))
# -------

sys.path.insert(0,"/var/www/catalog")

from application import app as application
```
6. Create a Virtual Environment and Install the Libraries Needed:
```
sudo apt-get install python3-pip
sudo -H pip3 install virtualenv
virtualenv env
source env/bin/activate
pip3 install -r requirements.txt
```
Install the following:
```
pip3 install flask packaging oauth2client redis passlib flask-httpauth
pip3 install sqlalchemy flask-sqlalchemy psycopg2 bleach requests
pip install oauthlib
pip install Flask-Login
pip install pyopenssl
```
#### 10. Configure apache server
```
sudo nano /etc/apache2/sites-enabled/catalog.conf
```
Then add the following content:
```
<VirtualHost *:80>
  ServerName <IP_Address or Domain>
  ServerAlias <DNS>
  ServerAdmin <Email>
  DocumentRoot /var/www/catalog
  WSGIDaemonProcess catalog user=grader group=grader
  WSGIScriptAlias / /var/www/catalog/catalog.wsgi

  <Directory /var/www/catalog>
    WSGIProcessGroup catalog
    WSGIApplicationGroup %{GLOBAL}
    Require all granted
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error.log
  LogLevel warn
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

#### 11. Reload & Restart Apache Server:
`sudo service apache2 reload`
`sudo service apache2 restart`

### Resources:
- [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
- [https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)
- [https://www.digitalocean.com/community/tutorials/how-to-use-sftp-to-securely-transfer-files-with-a-remote-server](https://www.digitalocean.com/community/tutorials/how-to-use-sftp-to-securely-transfer-files-with-a-remote-server)
- [https://www.codementor.io/@abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft](https://www.codementor.io/@abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft)
- [https://github.com/AliMahmoud7/linux-server-configuration/tree/642be30054376ea4ea5676ce455da0d684548f40](https://github.com/AliMahmoud7/linux-server-configuration/tree/642be30054376ea4ea5676ce455da0d684548f40)
- [https://aws.amazon.com/premiumsupport/knowledge-center/connect-http-https-ec2/](https://aws.amazon.com/premiumsupport/knowledge-center/connect-http-https-ec2/)
## An Important Note:
- After creating the EC2 instance, don't forget to edit the port settings on the website so that port 2200 and port 80 are open to requests.
