# Seven And A Half

sevenandahalf is a web app that takes your location and shows you the USGS topographic maps that you are located on.

It also includes a script to read a .csv file of maps, specify which ones you want using a year cutoff, a US State list, or a bounding box, download the maps, and initialize the app database with that data to allow you to access the maps.

To run, this app requires an apache2 webserver configured to use the WSGI standard. Instructions and tips are included for mod_wsgi.

_note that the following instructions can, if you pay attention to your own filepaths and configurations, go almost anywhere, but for ease (and to remind myself) I will specify system locations_

## Install Apache2
More complete details can be found at <https://www.raspberrypi.org/documentation/remote-access/web-server/apache.md>
bash:
```
sudo apt install apache2-dev -y
```

## Initial setup for the app
## Create a virtual environment in which to install the app:
Do this inside /usr/local/www/, creating the www directory if it does not exist already. Note that you may have to `sudo` to perform this command.
```
python3 -m venv venv
```

### Make your user the owner of the virtual environment to allow you to perform the later commands

```
sudo chownsudo chown $USER:$USER -R venv/
```
## Activate the environment you just created:

```
source venv/bin/activate
```
## Install the app into the virtual environment:

```
pip install https://charliemacquarie.com/software/sevenandahalf/dist/sevenandahalf-1.2.0-py3-none-any.whl
```

## Install and configure mod_wsgi
mod_wsgi is very configurable, and complete instructions are found in their docs:
- to install mod_wsgi: https://modwsgi.readthedocs.io/en/master/user-guides/quick-installation-guide.html
- to configure mod_wsgi: https://modwsgi.readthedocs.io/en/master/user-guides/quick-configuration-guide.html
- to note using virtual environments with mod_wsgi (necessary for a flask app): https://modwsgi.readthedocs.io/en/master/user-guides/virtual-environments.html
### download and unpack the source code
Note that the numbers in the release file may change depending on which release you are seeking.
```
wget https://github.com/GrahamDumpleton/mod_wsgi/archive/refs/tags/4.9.0.tar.gz
tar xvfz 4.9.0.tar.gz
```

### Configure the installation

change into the directory for the source code
```
cd mod_wsgi-4.9.0/
```
run the configuration script
```
./configure
```
### Complete the installation

```
make
sudo make install
```
### Activate the module
You have to create the correct load file in the apache2 configurations, for Rasberrypi this is usually at /etc/apache2/mods-available/. The file should be called wsgi.load, so:
```
sudo nano /etc/apache2/mods-available/wsgi.load
```
and add the single line `LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so` to the file.

Then, the module must be activated with:
```
sudo a2enmod wsgi
```
And finally, apache2 should be restarted for the changes to take effect:
```
systemctl restart apache2
```
At this point it may be wise to check the apache error log (usually at /var/log/apache2/error.log) to verify things appear to be working correctly

### Configure the site to use mod_wsgi and your wsgi script
Your default site configuration needs to be edited to add the Alias for the wsgi script (where the app will be imported) and give apache access to read the files in this location.
*** Should really just add in complete config for default-ssl.conf as appendix at end of this doc.
```
need to add in the configuration as it should be
```
You should also configure Daemon Process mode
```
need to add in the configuration as it should be
```


### Create the wsgi script the site will use to serve sevenandahalf
(create the /usr/local/www/sevenandahalf directory if it doesn't exist already)
```
sudo nano /usr/local/www/sevenandahalf/where.wsgi
```
and inside this file add the following lines:
```
import sevenandahalf

application = sevenandahalf.create_app()
```
Restart apache again for the changes to take effect:
```
systemctl restart apache2
```
*** Next up: configure daemon process mode, setup default-ssl.conf correctly, make the fake cert work, and (I think?) chown to $USER for the document root.
## Setup apache ssl (to enable https://)
### Enable the ssl module and the default-ssl site:
```
sudo a2enmod ssl
sudo a2enmod rewrite
sudo a2ensite default-ssl
```
(restart for the changes to take effect)
```
systemctl restart apache2
```



## Setup sevenandahalf!
### Tell the system what/where the flask app is to use the setup processes:

```
export FLASK_APP=sevenandahalf
```

### Get a list of all the USGS topographic maps to use to initialize your sevenandahalf
```
wget https://charliemacquarie.com/librarystorage/other/progress/allthemaps/topomaps_all.zip
unzip topomaps_all.zip
```
the resulting csv file will be the file you should use with the get-maps command.

### Make your user the owner of the web-root so that the necessary folders can be created by script

```
sudo chown -R $USER:$USER /var/www/html/
```

### Download some maps with the get-maps command
Note that depending on what you decide to download, this command may take a very long time to run.
```
flask get-maps --web-root /var/www/html/ topomaps_all.csv
```

### Initialize the database with the maps you downloaded
```
flask init-db
```

### Your site should now be ready to visit!

192.168.1.153 in a browser on my network, yours may vary!

# Setup specific to Trucknet
(<https://github.com/charliemacquarie/trucknet>)
At the document root (/var/www/html/), delete the default index.html from Apache2:
```
sudo rm -rf index.html
```
Still at the document root (/var/www/html/), clone the trucknet repository into place:
```
sudo git clone https://github.com/charliemacquarie/trucknet .
```
Edit the main apache2 config file to allow .htaccess override for directory styling. In /etc/apache2/apache2.conf, under `<Directory "/var/www">`, replace `AllowOverride None` with `AllowOverride All`.
