# Project: Linux Server Configuation
A part of the Udacity Full Stack Developer Nanodegree. This project involves the creation of an Ubuntu server on Amazon Lightsail and configuring it to server a flask app running a PostgreSQL database.

## Key info
Server IP: ```35.163.196.197```

## Setting up the Lightsail server.
1. Go to [Amazon Lightsail](https://lightsail.aws.amazon.com).
2. Generate a public/private key pair locally in your terminal with ```ssh-keygen```. Save the private key in an appropriate folder (e.g. ```~/.ssh/```). Protect the private key with a password.
3. In the Lightsail interface, create an instance, and choose the option to upload the public key you just generated.
4. After the Lightsail server is booted up, create and attach a static IP to your server instance. For me, this is ```35.163.196.197```.

## Configuring the Server

1. Add the private key to your ssh-agent by running in the terminal.
    ```
    eval `ssh-agent```
    ssh-add ~/.ssh/private_key # Your private key location
    ```
2. SSH into the Lightsail server (ubuntu is a default user).
    ```
    ssh ubuntu@35.163.196.197
    ```
3. Update all installed packages.
    ```
    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get autoremove
    ```
4. Change the ssh port from 22 to 2200 by ```sudo vim /etc/ssh/sshd_config``` and uncommenting and changing ```#Port 22``` to ```Port 2200```. This provides some security through obscurity. Then run ```sudo service sshd restart```.

5. Disconnect from your ssh connection. Then inside the Manage page of your Lightsail instance, under Networking and remove the SSH Port 22 rule and add a TCP Port 2200 rule. Then ssh in again, but now with a ```-p 2200``` flag.

6. Configure the Uncomplicate Firewall (ufw) to only allow SSH on port 2200, HTTP on port 80, and NTP (network time protocol) on port 123.
    ```
    sudo ufw status
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp  # For SSH
    sudo ufw allow http  # Alias for 80/tcp
    sudo ufw allow ntp  # Alias for 123/udp
    sudo ufw enable
    sudo ufw status
    ```
7. Check that in ```/etc/ssh/sshd_config``` the line ```PasswordAuthentication``` is ```no``` to force token based authentication.
8. Additionally, change ```PermitRootLogin``` to ```no``` to prevent sshing as root. Restart ssh with ```sudo service sshd restart```.

9. Check that that the time zone is UTC with by running ```timedatectl```.

## Creating a user for the grader

1. ```sudo apt-get install finger``` for conveniently getting user info.
2. ``` sudo adduser grader``` to make a new user for the grader. Set the password, but you shouldn't need to pass this password to anyone as we are using token-based auth only.
3. Give the grader sudo privileges.
    ```
    cd /etc
    sudo cp sudoers.d/90-cloud-init-users sudoers.d/grader
    sudo nano sudoers.d/grader  # Vim can't get write permissions here
    ```
    And edit the ubuntu user line to ```grader ALL=(ALL) NOPASSWD:ALL```

4. Open a second terminal on the local machine and generate another public/private key pair with ```ssh-keygen```.

5. In the Lightsail server, go to ```/home/grader```, make a new ```sudo mkdir ./ssh``` folder and ```sudo nano ./ssh/authorized_keys```. In this file, paste the newly created public key.

6. Protect these keys from other user modification.
    ```
    sudo chmod 700 .ssh
    sudo chmod 644 .ssh/authorized_keys
    ```
7. Change .ssh owner to grader
    ```
    sudo chown -R grader:grader .ssh
    ```
8. Restart the ssh service with ```sudo service sshd restart```.
9. Now, in your local machine, ssh into the grader account with the grader private key.
    ```
    ssh -v -p 2200 grader@35.163.196.197 -i ~/.ssh/FSNDgrader
    ```

## Preparing to deploy Wildsight Flask app
1. Install apache2 and the mod-wsgi interface between python3 and apache2.

    ```
    sudo apt-get install apache2
    sudo apt-get install libapache2-mod-wsgi-py3
    ```
2. Install pip for python3
    ```
    sudo apt-get install python3-pip
    ```
3. Install PostgreSQL
    ```
    sudo apt-get install postgresql
    # required for PSQL to function w/ apache/python
    sudo apt-get install libpq-dev
    sudo apt-get install postgresql-contrib
    ```
4. Configure PostgreSQL to not allow remote connections with ```sudo less /etc/postgresql/10/main/pg_hba.conf```. The file should be similar to below. See https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps for more info.

    ```
    # "local" is for Unix domain socket connections only
    local   all             all                                     peer
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            md5
    # IPv6 local connections:
    host    all             all             ::1/128                 md5
    # Allow replication connections from localhost, by a user with the
    # replication privilege.
    local   replication     all                                     peer
    host    replication     all             127.0.0.1/32            md5
    host    replication     all             ::1/128                 md5
    ```
5. Create a PSQL database user in order to limit database access. First, log into the ```postgres``` linux user created by the PSQL installation.


    ```
    # -u is for user, -i is for login (i.e. reads profile etc.)
    sudo -u postgres -i
    ```

    Start the psql command line with ```psql```. Then:

    ```
    /* Creates user with a password, and db creation powers */
    CREATE USER catalog WITH PASSWORD 'not_the_real_password' CREATEDB;
    /* Create a db named catalog */
    CREATE DATABASE catalog WITH OWNER catalog;
    ```

    Then limit the catalog db rights to only the user catalog.
    ```
    REVOKE ALL ON SCHEMA public FROM public;
    GRANT ALL ON SCHEMA public TO catalog;
    ```

    Exit out of the psql command line and postgres. Then restart the psql service:
    ```
    sudo service postgresql restart
    ```
## Deploying the Wildsight Flask app
This part gave me some trouble because it seemed there was some differences between how files could be found depending on whether I ran them using ```python3``` or whether they were being run through apache2. So some of the things I do here are to work around this, but I'll point them out as I go along.

1. Install the app into ```/var/www/``` and rename it to ```wildsight```.
    ```
    # While in /var/www/
    sudo git clone https://github.com/andrewKOwong/FSNDwildsight
    sudo mv FSNDwildsight wildsight
    ```

2. Install python3 packages required for the app. I originally developed this app in a vagrant VM environment that is pretty outdated, so I rather that installing from ```pip freeze > requirements.txt``` file that contained out-of-date packages, I made a separate ```lightsail_requirements.txt``` in the ```wildsight``` folder that installs new supported versions of packages I used in import statements. I did this manually, and this could be improved in the future. The file looks like this:

    ```
    ## lightsail_requirements.txt
    # Used in wildsight.py
    Flask-SQLAlchemy
    SQLAlchemy
    Flask
    oauth2client
    httplib2
    requests
    # Additionally used in database_setup.py
    # None
    # Additionally used in populate_db.py
    pandas
    # Additionally used in sighting_form.py
    Flask-WTF
    WTForms
    # Used by SQLAlchemy by not imported directly
    psycopg2
    ```

    Installing these packages with ```sudo pip3 install``` puts the packages into ```/home/ubuntu/.local/lib/python3.6/site-packages```, which I found isn't accessible when deploying with ```apache2```. I might be able to avoid this problem with ```virtualenv```, but to keep things simple, I decided to install packages globally instead.

    ```
    # Note the -H flag
    sudo -H pip3 install -r wildsight/lightsail_requirements.txt
    ```
3. Add a empty init file into ```/var/www/wildsight``` to allow it to be recognizable as a python package.
    ```
    sudo touch wildsight/__init__.py
    ```
4. Make a ```wildsight.wsgi``` file that loads the wildsight app. This is used by ```apache2``` to start the app. Critically, the flask ```app``` object in ```wildsight.py``` (actually ```wildsight_lightsail.py```, see below) needs be imported as ```application```. This file should look like this:

    ```
    #!/usr/bin/env python3
    import logging
    import sys
    import os

    # /var/log/apache2/error.log captures stderr from the server
    # Send logging to stderr
    logging.basicConfig(stream=sys.stderr)

    # Printing to stderr for debugging
    # See https://stackoverflow.com/questions/5574702/how-to-print-to-stderr-in-python
    def eprint(*args, **kwargs):
        print(*args, file=sys.stderr, **kwargs)

    # When Apache2 runs, for w/e reason it doesn't have a import 
    # finder that locates .py files in the same directory.
    # Therefore, add the absolute path of the flask app.
    app_directory = '/var/www'
    if sys.path[0] != app_directory:
        sys.path.insert(0, app_directory)

    # Similarly for the actual wildsight directory
    inner_app_directory = '/var/www/wildsight'
    if sys.path[0] != inner_app_directory:
            sys.path.insert(0, inner_app_directory)

    from wildsight.wildsight import app as application
    application.secret_key = os.urandom(32)
    ```
    Worth noting here is that I am ensuring that logging is going to ```stderr```, which is captured in ```/var/log/apache2/error.log```. The ```eprint()``` function is also handy for printing to ```stderr``` during debugging.

    When running python scripts using ```python3```, ```import``` statements seem to be able find modules in the same directory as the script, whereas when the script is run by ```apache2```, this doesn't seem to be the case. This may be related to the fact that ```os.getcwd()``` returns ```/var/www``` when ```wildsight.wsgi``` is run by ```python3```, whereas it returns ```/``` when run by ```apache2```. In any case, directly inserting ```/var/www``` and ```/var/www/wildsight``` into ```sys.path``` solves this problem.

5. Switch to PostgreSQL as the database. The original wildsight app used a SQLite3 database. I first made copies of all the python files that access the database, and removed the old ones, so that I can cleanly keep the old files around, either on the same git branch or a new one. 


    ```
    sudo cp wildsight/wildsight.py wildsight/wildsight_lightsail.py
    sudo cp wildsight/database_setup.py database_setup_lightsail.py
    sudo cp wildsight/populate_db.py populate_db_lightsail.py
    sudo rm wildsight/wildsight.py
    sudo rm wildsight/database_setup.py
    sudo rm wildsight/populate_db.py
    ```

    Then I changed the import references in these files from ```database_setup``` to ```database_setup_lightsail```.

    Next I change SQL engine references from sqlite to postgresql. Note the reference to the ```catalog``` user, and replace [password] with the real ```catalog``` user password.:
    ```
    # In wildsight_lightsail.py
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///wildsight.db'
    # Becomes
    app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://catalog:[password]@localhost/catalog'

    # In database_setup_light.py and populate_database_lightsail.py
    engine = create_engine('sqlite:///wildsight.db')
    # Becomes
    engine = create_engine('postgresql://catalog:[password]@localhost/catalog')
    ```

6. Set up and populate the database the PSQL database.
    ```
    sudo python3 wildsight/database_setup_lightsail.py 
    sudo python3 wildsight/populate_db_lightsail.py 
    ```

7. Set up ```apache2``` to load the Wildsight app. First add a new file ```/etc/apache2/sites-available/wildsight.conf```
    ```
    <VirtualHost *:80>

            ServerName 35.163.196.197
            WSGIScriptAlias / /var/www/wildsight.wsgi

            <Directory /var/www/wildsight/>
                    Require all granted
            </Directory>
            ErrorLog ${APACHE_LOG_DIR}/error.log
            LogLevel warn
            CustomLog ${APACHE_LOG_DIR}/access.log combined

    </VirtualHost>
    ```

    Then, enable the site, and restart apache.
    ```
    sudo a2ensite wildsight.conf
    sudo systemctl reload apache2
    ```

    Disable the default site
    ```
    sudo a2dissite 000-default.conf
    sudo systemctl reload apache2
    ```
8. Add the Google signin client secrets file into the ```wildsight``` folder. Do this by using ```scp```. Also change the reference to it in ```wildsight_lightsail.py``` to ```/var/www/wildsight/DO_NOT_COMMIT_client_secrets.json```, to be able to find it when running with ```apache2```.

9. Restart the ```apache2``` server with ```systemctl reload apache2```. If you now visit http://35.163.196.197, you should be able to see the app!

    Note that Google Signin doesn't work because I don't have a top level private domain that I can authorize with Google (e.g. yadayada.com).


# ACKNOWLEDGEMENTS

I used https://www.codementor.io/abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft when I was deploying a minimal python test app to troubleshoot my deployment.

I had some trouble with the PSQL part, and I referenced several previous Udacity student's projects to figure it out:  
https://github.com/juvers/Linux-Configuration  
https://github.com/twhetzel/ud299-nd-linux-server-configuration  
https://github.com/boisalai/udacity-linux-server-configuration

as well as the following resources:

https://www.postgresql.org/docs/8.0/sql-createuser.html
https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps

I referenced this for changing the ssh port:  
https://ca.godaddy.com/help/changing-the-ssh-port-for-your-linux-server-7306

