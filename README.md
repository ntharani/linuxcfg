# UDACITY FSD PROJECT #5 - LINUX SERVER CONFIGURATION

## <TL;DR>
- sh -i <location of awslightsail_udacity key>  grader@54.71.17.44 -p 2200
- [http://54.71.17.44](http://54.71.17.44) (If I'm still paying for it)
- Started Work. Drank Tea. Drank more tea. Debugged. Learned A lot. Had more tea. Read Blogs.
- Contemplated life. Read more stack overflow and other tech blogs. Got it working.
- Oh look... a puppy...

## Objective: 
- Take Item Catalog Flask Web App (Project #3/#4: DrinksDB) and host it on a secured Ubuntu box hosted on AWS Lightsail.
- It is recommended that the Item Catalog Project database be switched from SQLite3 to Postgres.

## Approach:

1. Verify DrinkDB works with with Postgres locally on a clean vagrant box.
2. Procure and securely configure AWS Lightsail box with Ubuntu 16.04 LTS, Postgres and all other app requirements to host it.
3. Test and configure DrinksDB on hosted platform

## AWS Lightsail Setup & Config Basics

1. Setup AWS LightSail. Setup Static IP and bind it to the box.
2. Setup AWS Firewall per specs: Add port TCP 2200. Download the SSH Keys from Lightsail.
3. Use AWS Console to SSH in to Ubuntu box (default on port 22 still)
4. Upgrade the box:
	- `sudo apt-get update`
	- `sudo apt-get upgrade`
	- `sudo apt-get autoremove`
	- `sudo apt-get install finger`
	- if some packages still show as available for upgrade, you may need to also do `sudo apt-get dist-upgrade`
	- Handy command to check for a package: `sudo apt list --installed | grep -i apache`
	- Verify time is UTC. It is by default on this. To configure: `sudo dpkg-reconfigure tzdata`

## Accessing Server via SSH
- Reference the SSH file you downloaded and verify you can login to the instance
- `ssh -i ~/Downloads/LightsailDefaultPrivateKey-us-west-2.pem ubuntu@57.71.17.44`
- It might not work, and you may get this error:

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/Users/nagib/Downloads/LightsailDefaultPrivateKey-us-west-2.pem' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "/Users/nagib/Downloads/LightsailDefaultPrivateKey-us-west-2.pem": bad permissions
Permission denied (publickey).
- fix it: `chmod 600 LightsailDefaultPrivateKey-us-west-2.pem`
- You should be logged in at this point.

## Setup Firewall

- `sudo ufw status`
- `sudo ufw default deny incoming`
- `sudo uff default allow outgoing`
- `sudo uff status`
- `sudo ufw allow ssh` (default), but in our case `sudo ufw allow 2200/tcp` (Non standard port)
- `sudo ufw allow www`
- `sudo ufw allow ntp`
- You can't see the configuration until you enable it. `sudo ufw enable`
- `sudo ufw status -verbose`

## User Accounts

- On install: default user on Ubuntu is ubuntu.
- `sudo adduser grader` - be sure to add a password
- Try logging in as grader:
	- 	ssh grader@127.0.0.1 (it will fail, because Ubuntu setup does not allow password logins by default)

### Assign SUDO Permissions

- `/etc/sudoers` >>  This file can be overwritten.
- Ubuntu is different it references `/etc/sudoers.d` from above: `sudo ls /et/sudoers.d`
- Don't create this file manually, use visudo.
- `sudo visudo -f /etc/sudoers.d/<somefile>`
- In the file, set `<user> ALL=(ALL:ALL) ALL`

References:
- https://unix.stackexchange.com/questions/122087/what-is-the-best-way-to-add-a-user-to-the-sudoer-group
- https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file-on-ubuntu-and-centos

### SSH Key Pair Generation for new ubuntu user

#### On the Ubuntu Box
You won't be able to login as password authentication has been disabled.
- `sudo nano /etc/ssh/sshd_config` and ensure `PasswordAuthentication` is `NO`.
- (It's the default on lightsail. For initial configuration however, you will need to temporairily change it to yes)
- Disable Root Login. Make sure `PermitRootLogin no` is in the file.
- Update SSH & SSHD
	- `sudo service ssh restart`
	- `sudo service sshd restart`	
- Note: We haven't yet told SSHD to listen on port 2200. I know you're anxious. Sit tight. :) 

#### On your local machine:
- `ssh-keygen`. Follow instructions. Name it something useful.
- This generates two files: file and file.pub.
- The AWS key you downloaded from lightsail will be required here:

#### Transfering Public Key to New Server


- login as <target user>, eg: grader: ssh grader@127.0.0.1. (Do this after you have SSH'd in as user ubuntu). 

Hard Way:

- `mkdir .ssh`
- `touch .ssh/authorized_keys`
- `nano .ssh/authorized_keys`
- `paste grader.pub`
- `chmod 700 .ssh`
- `chmod 644 .ssh/authorized_keys`

Easy Way:

- use ssh-copy-id: 
- Reference: https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys

Now logout and try again via SSH, you should not need a password login anymore. Which means:

#### Changing SSH Default Port (if desired) and set PasswordAuthentication back to NO

- `sudo nano /etc/ssh/sshd_config` and ensure `PasswordAuthentication` is `NO`.
- Change the SSH port, if desired, eg: 2200
	- `sudo service ssh restart`
	- `sudo service sshd restart`	

### Set up a Snapshot through AWS console. (Optional)

## Server Setup to Host Flask WebApp

### AWS LightSail Essential Packages 
- `sudo apt-get install postgresql`
- git (usually installed by default)

### Postgres Configuration
- create a new database user named `catalog` with limited permissions to your db.
- `sudo su postgres`
- `psql`
- here: `Create USER <db user name> nagib with password '<password>'`
- This will allow you to create an engine command w/ SQLAlchemy of the form:
- `engine = create_engine('postgresql://catalog:<password>@localhost/ndrinks')`


#### Nifty Postgres Commands

- postgres=# \l

                                  List of databases
|  Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   |
---------- |----------| ---------| ------------|-------------|------------------------
 forum     | vagrant  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 news      | vagrant  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |

 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 vagrant   | vagrant  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(6 rows)


- postgres=# `SELECT rolname FROM pg_roles;`
- `psql -d ndrinks` (Open a particular DB)
- `\dd`
- `\dt`
- `\q`
- `select * from information_schema.tables;`
- `\conninfo`

|rollname|
----------
 postgres	|
 vagrant | 
 catalog	|	
(3 rows)	|

#### DB Screw-ups and how to fix:
- You may mess things up. I certainly did. Which means nuking the DB and restarting.
- `sudo su postgres`
- `psql`
- DROP DATABASE <database name>
- CREATE DATABASE <database name>

### Flask, Apache and WSGI.

Get a basic Flask WebApp working. **Don't worry about your app until you have a basic one running.**

References:
- https://devops.profitbricks.com/tutorials/install-and-configure-mod_wsgi-on-ubuntu-1604-1/
- http://initd.org/psycopg/docs/
- https://packages.debian.org/sid/libpq-dev
- https://www.theodo.fr/blog/2017/03/developping-a-flask-web-app-with-a-postresql-database-making-all-the-possible-errors/
- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- https://github.com/PiJoules/FlaskApache (I followed this one)

#### /var/www
- This is where your WebApp app will live.
- TL:DR; 
	1. Create folder name for the WebApp. 
	2. In that folder, git clone your repo. I named it the samething per the example, but not necessary. 
	3. In the same parent. Create a wsgi file for this. 
	4. Create a file in /etc/apache2/site-available with the same name parent folder, eg: FlaskApache.conf
	4. Restart Apache. `sudo /etc/apache2/sites-available/FlaskApache.conf`
	5. Tail the apache log. You'll probably discover a bunch of stuff you need to fix/refactor in your app later.

### Git
- Go grab your project. Don't worry about SSH here. Use HTTPS for git clone. Clone the repo under the Flask Project name. (Eg: /var/www/FlaskApache/<git cloned repo>
- In my case, I had made changes to accomodate postgres and general refactoring under a new branch remote called postgresv2. You can close a branch version (and rename it) by doing the following:
- git clone <repo url> --branchname <rename of repo on clone>. In my case, FlaskApache 

### Learn About PIP and VirtualENV (*)
- https://www.dabapps.com/blog/introduction-to-pip-and-virtualenv-python/
- https://springmerchant.com/bigcommerce/psycopg2-virtualenv-install-pg_config-executable-not-found/
- ProTip: If you need to sudo anything while in a virtualEnv, take it as a warning of problems to come. You can usually fix 90% of the issues by making sure that virtual env directory has appropriate permissions.

  497  `sudo virtualenv env`
  498  `sudo chmod -R 777 env`
  499  `source env/bin/activate`
  500  `pip install Flask`
  504  `sudo cp application.py __init__.py`
  505  python __init__.py #does it work??
  506  pip install -r requirements.txt
  507  python __init__.py #does it work??
  509  `sudo apt install libpq-dev python-dev`
  510  python __init__.py #does it work??
  514  `pip install psycopg2` 
  515  python __init__.py #does it work??
  516  `sudo service apache2 restart`
  517  `sudo tail -f /var/log/apache2/error.log`
  521  sudo chmod u+x FlaskApache.wsgi #possibly necessary
  522  sudo service apache2 restart
  
  later, rinse, repeat. Have some tea.

### DrinksDB (Project #3/#4) Refactor
- Figure out what your dependencies are. 3 options:
	1. Be organized upfront. Meh. But if you were, then hopefully you setup a python virtual environment and then you  can just issue the following command: `pip freeze > requirements.txt`
	2. Use pipreqs, or pigar: https://github.com/Damnever/pigar. (I used pigar). This generated my requirements.txt file. Woot.
	3. Learn - the hardway (*) - that requirements.txt isn't enough. It'll miss shit like psycopg2. Deliberately.

- Look for 3 files:
	- Usally a DB setup file (may be sqlite, you need to chage the engine if using SQLAlchemy)
	- A DB file to populate the the DB.
	- An application file. (eg: `application.py`) In all cases the engine must be changed to reflect the new one.
- Rename main application (eg: application.py) file to `__init__.py`
- In all files that reference the DB engine, change if necessary to postgresql:
	- `engine = create_engine('postgresql://catalog:<password>@localhost/ndrinks')`
- Keep debugging the error log until your launches: `sudo tail -f /var/log/apache2/error.log`

#### For the record
- Turns out I had hardcoded several localhost links in Project#3. Had to fix that.
	- https://stackoverflow.com/questions/11124940/creating-link-to-an-url-of-flask-app-in-jinja2-template
- Client Secrets.json --> Used Relative paths. Bad Idea. 
	- http://help.pythonanywhere.com/pages/NoSuchFileOrDirectory
- Social Login --> Wouldn't fire. Turns out I needed to copy app.secret from bottom and have it up close to where Flask was imported.
	- https://stackoverflow.com/questions/26080872/secret-key-not-set-in-flask-session
- My DB had sereral issues around backrefs and cascade issues that I needed to fix.
	- http://docs.sqlalchemy.org/en/rel_0_9/orm/backref.html#backref-arguments
	- https://stackoverflow.com/questions/5033547/sqlalchemy-cascade-delete

### Postgres Bonus
- You know that Users table you built in Project#3/#4?? The one with SQlite? The one that probably have no errors with queries but suddenly threw errors with Postgres? 
- ** USERS is a reserved word. Avoid naming your tables as such. You can get around it by quoting "users" in your queries, but it's messy. I learned a lot. I've also lost more than a few hours that I won't get back. But I won't make this mistake again.
- https://dba.stackexchange.com/questions/75551/returning-rows-in-postgresql-with-a-table-called-user

### Social Login (Project#3/#4)
- Chances are you used localhost, so you'll need to double back to google, facebook et al and add the IP of this server.

### In case you've read this far
- https://blog.appdynamics.com/engineering/a-performance-analysis-of-python-wsgi-servers-part-2/
- bjoern looks really interesting.

## Have a look. (Assuming I'm still paying for it): http://54.71.17.44


 