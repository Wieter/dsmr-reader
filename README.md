# DSMR-reader
*DSMR-protocol reader, telegram data storage and energy consumption visualizer. 
Can be used for reading the smart meter DSMR (Dutch Smart Meter Requirements) P1 port yourself at your home. 
You will need a cable and hardware that can run Linux software. 
**Free for non-commercial use**.*

This project is forked from https://github.com/dsmrreader/dsmr-reader

Note: this manual is still work in progress!

# Instructions for installation on FreeBSD
With minor modifications, they can be followed for most Linux distributions as well.

## Database server setup
Setting up the database is beyond the scope of this manual. Some already have a database server running elsewhere, some want it in a separate jail or vm, or an entirely different flavor of database-server.
As **PostgreSQL** is suggested and very well established, I highly recommend installing its latest version on a dedicated machine. 

Here you can find some nice manuals (not mine):
* for FreeBSD:
https://www.howtoforge.com/how-to-install-postgresql-on-freebsd-12/
* for Linux (e.g. rasbian on the Raspberry Pi): 
https://www.tecmint.com/install-postgresql-database-in-debian-10/

Note: Since Django can handle a [multitude of database server backends](https://docs.djangoproject.com/en/3.1/ref/databases/), just choose your preffered server and setup as you like.
Remember the following parameters (obviously, change them!): 
* hostname `DATABASE_HOST`
* a dedicated dsmr database `DATABASE_NAME` owned by ..
* a dedicated dsmr user `DATABASE_USER` using ..
* the password `DATABASE_USERPASSWORD`

## Webinterface setup
First, we will install the webinterface and its requirements.

### Install required packages for the Web Server Gateway Interface 
```shell
sudo pkg install nano nginx git uwsgi bash python3
```
~~(note to self: no more need for py37-psycopg2)~~

### Prepare configuration directories
```shell
sudo mkdir -p /usr/local/etc/uwsgi/vassals
sudo mkdir -p /var/log/uwsgi
```

### Enable the services
Modify /etc/rc.conf (alternatively use nano to manually edit this file): 
```shell
sudo sysrc uwsgi_enable=YES uwsgi_emperor=YES uwsgi_vassals_dir="/usr/local/etc/uwsgi/vassals" nginx_enable=YES
```

### Add user for backend without login permissions - for security reasons
```shell
sudo pw user add -n dsmr -d /usr/local/www/dsmr/ -G www -m -s /usr/sbin/nologin
```

### Create virtual environment for installing all python packages without disturbing the base system
```shell
sudo -u dsmr python3 -m venv /usr/local/www/dsmr/.venv
```
### Activate the virtual environment and jump into the home directory
```shell
sudo -u dsmr -- sh -c "cd /usr/local/www/dsmr/; /usr/local/bin/bash"
. .venv/bin/activate
```

***Ensure that the shell looks like something like this: (.venv) [dsmr@hostname /usr/local/www/dsmr]$!***

***Until mentioned otherwise, this should stay the case for all of the following instructions!***


### Obtain the project files
Clone the desired repository files:
```shell
git clone https://github.com/Wieter/dsmr-reader.git ~/dsmr-reader
```

### Configure the webinterface
Jump directory and copy a settings template:
```shell
cd dsmr-reader
cp dsmrreader/provisioning/django/settings.py.template dsmrreader/settings.py
```

Note: For FreeBSD installations, the pg_dump command for backing up the database is not automatically found using the current configuration. Therefore it is advisory to set the full path:
```shell
echo "\nDSMRREADER_BACKUP_PG_DUMP = '/usr/local/bin/pg_dump'" >> dsmrreader/settings.py
```

### Configure the environment
Copy an environment template:
```shell
cp .env.template .env
```
And generate a secret key automagically:
```shell
./tools/generate-secret-key.sh
```

### Configure the database information
In the following section, edit the parameters for `DJANGO_DATABASE_HOST`, `DJANGO_DATABASE_NAME`, `DJANGO_DATABASE_USER` and `DJANGO_DATABASE_PASSWORD` to whatever you have set in the first step (Database server setup), for example using the `nano` editor.

* Execute:
```shell
nano .env
```

If a different database-server than PostgreSQL is used, edit also `DJANGO_DATABASE_ENGINE` and `DJANGO_DATABASE_PORT` accordingly. 

This is also a good time to setup an username/password for the web super user. To do so, set the parameters for `DSMRREADER_ADMIN_USER` and `DSMRREADER_ADMIN_PASSWORD` to your liking. 

Also, ensure a static root location is present, for static files (e.g. css/js/images): 
```
DJANGO_STATIC_ROOT=/usr/local/www/dsmr/dsmr-reader/static
```

### Install python dependencies
Install some dependencies as required by the project:
* Execute:
```shell
pip3 install -U pip
pip3 install -r dsmrreader/provisioning/requirements/base.txt
pip3 install psycopg2-binary
```

(note to self: remove pyserial later on, this is not needed in the webinterface as we will solely rely on the API)

### Bootstrapping
Now it is time to bootstrap the application and check whether all settings are good and requirements are met.

* Execute this to initialize the database we’ve created earlier:
```shell
./manage.py migrate
```

Prepare static files for webinterface. This will copy all static files to the directory we created for Nginx earlier in the process. It allows us to have Nginx serve static files outside our project/code root.
* Sync static files:
```shell
./manage.py collectstatic --noinput
```

Create an application superuser with the following command. The `DSMR_USER` and `DSMR_PASSWORD` as defined in Env Settings (.env file) will be used for the credentials.
* Execute:
```shell
./manage.py dsmr_superuser
```

### Exit the virtual environment
**we should now exit the virtual environment**

* Execute:
```shell
exit
```

* Ensure the shell returns to normal:
```shell
user@hostname:/usr/local/www/dsmr %
```

### Copy the wsgi and webserver configuration files
* Execute:
```shell
sudo cp /usr/local/www/dsmr/dsmr-reader/dsmrreader/provisioning/uwsgi/dsmr-reader_uwsgi.ini /usr/local/etc/uwsgi/vassals/
```

#### If using NGINX webserver (as suggested):
* Execute:
```shell
sudo mkdir /usr/local/etc/nginx/servers/
```
copy the configuration:
* Execute:
```shell
sudo cp /usr/local/www/dsmr/dsmr-reader/dsmrreader/provisioning/nginx/dsmr-reader.conf /usr/local/etc/nginx/servers/
```
and edit nginx.conf: 
* Execute:
```shell
sudo nano /usr/local/etc/nginx/nginx.conf
```
to include at the end of the http section:
```
http {
  ...
  include "servers/*.conf";
}
```
#### If using Apache24 webserver (an alterative):
Make sure it is installed and enabled (instead of nginx).
* Execute:
```shell
pkg install apache24
sudo sysrc apache24_enable=YES
```
Copy the configuration:
* Execute:
```shell
sudo cp /usr/local/www/dsmr/dsmr-reader/dsmrreader/provisioning/apache/dsmr-reader.conf /usr/local/etc/apache24/Includes/
```

## Start and test the webinterface 
### Start services
* Execute:
```shell
sudo service uwsgi start
sudo service nginx start
```
If using apache, run `sudo service apache24 start` instead of nginx.

### Browse webinterface
Verify if everything is running nicely, by going with a browser to the webinterface:
```
http://ip.address.of.the.webinterface:8000
```

A very nice dashboard should appear.

### Setup the API target
Browse to:
```
http://ip.address.of.the.host:8000/admin/dsmr_api/apisettings/1/change/
```
and 
* enable API-calls. 
* Note down the API Key somewhere for later usage. 
**Don't forget to SAVE!**

## Setup the data logger
### Connect hardware
Firstly, connect a P1->USB cable to your target hardware.


### Create dedicated user
**If using the same host as above, you can skip the user and venv creation, and go directly to the next step: 'Enter the virtual environment'.**

Create a dedicated user
```shell
sudo pw user add -n dsmr -d /usr/local/www/dsmr/ -G dialer -m -s /usr/sbin/nologin
```

Install python and create virtual environment
`sudo pkg install bash python3`
`sudo -u dsmr python3 -m venv /usr/local/www/dsmr/.venv`

### Activate the virtual environment
If you jumped to here because of using the same host as the webinterface, firstly ensure proper group permissions:
* add dsmr user to dialer group: 

```shell
pw groupmod dialer -m dsmr
```

Activate the virtual environment:
```shell
sudo -u dsmr -- sh -c "cd /usr/local/www/dsmr/; /usr/local/bin/bash"
. .venv/bin/activate
```

***Ensure that the shell looks like something like this: (.venv) [dsmr@hostname /usr/local/www/dsmr]$!***

***Until mentioned otherwise, this should stay the case for all of the following instructions!***

Install some dependencies as required by the project:
* Execute:
```shell
pip3 install -U pip
pip3 install install pyserial==3.4 requests==2.24.0 python-decouple==3.3
```

Create an environment file containing info about where to sent the data to
```shell
nano .env
```
containing the following(adjust the parameters!):
```
### The DSMR-reader API('s) to forward telegrams to:
DATALOGGER_API_HOSTS=http://ip.address.of.the.webinterface:8000
DATALOGGER_API_KEYS=APIKEY_AS_FOUND_BEFORE

### Input method
DATALOGGER_INPUT_METHOD=serial
DATALOGGER_SERIAL_PORT=/dev/cuaU0
DATALOGGER_SERIAL_BAUDRATE=115200

### Sleep settings
DATALOGGER_SLEEP 10
```

**Note: Comment out (by prepending #) DATALOGGER_SLEEP to use the default settings (every second). DSMR5 meters send a telegram every 1s, but this can be considered too much.**

Note: for linux on the raspberry pi, the usb serial is likely to be: `/dev/ttyUSB0`

Obtain the script for gathering the data:

```shell
wget -O ~/dsmr_datalogger_api_client.py https://raw.githubusercontent.com/Wieter/dsmr-reader/development-lite/dsmr_datalogger/scripts/dsmr_datalogger_api_client.py
```

### Exit the virtual environment
**we should now exit the virtual environment**

* Execute:
```shell
exit
```

* Ensure the shell returns to normal:
```shell
user@hostname:/usr/local/www/dsmr %
```

### Setup daemon for data gathering
Copy the daemon configuration file and ensure it is executable:
```shell
sudo mkdir /usr/local/etc/rc.d
sudo cp /usr/local/www/dsmr/dsmr-reader/dsmrreader/provisioning/daemon/dsmr_datalogger /usr/local/etc/rc.d
sudo chmod +x /usr/local/etc/rc.d/dsmr_datalogger
```
And enable it:
`sudo sysrc dsmr_datalogger_enable=YES`
And start it:
`sudo service dsmr_datalogger start` 

Note: For linux, use a systemd configuration file. 

(note to self: prepare systemd file example)

(note to self: setup proper logging in the daemon configuration file)

## Thats it! Now go back to your dashboard and watch the data flowing in :)
