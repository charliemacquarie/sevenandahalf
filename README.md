# Seven And A Half

sevenandahalf is a web app that uses your location to show you the USGS topographic maps covering where you are. It is built using [Flask](https://flask.palletsprojects.com/en/2.1.x/)

sevenandahalf is designed to work on a local network that's not connected to the internet. Specifically, the idea is that you can set this up on something like a Rasberrypi which can manage its own wifi network, and then users can access maps (and other stuff too, if you setup trucknet) using their gps location -- all without needing to rely on the internet (because you're probably somewhere without the internet, right?).

Obviously, this use case also requires that you figure out someway to power the server portably, but you're on your own for that.

sevenandahalf also includes a script to read a .csv file of maps, specify which ones you want using a year cutoff, a US State list, or a bounding box, download the maps, and initialize the app database with that data to allow you to access the maps.

To run, this app requires an apache2 webserver configured to use the WSGI standard. Instructions and tips are included for mod_wsgi. If you are familiar with configuring both of these, you may be able to set up sevenandahalf without the highly-detailed instructions provided below. They're mostly here so that I don't forget every time I have to do it.

_note that the following instructions can, if you pay attention to your own filepaths and configurations, go almost anywhere, but for ease (and to remind myself) I will specify system locations_

## Install Apache2
More complete details can be found at <https://www.raspberrypi.org/documentation/remote-access/web-server/apache.md>
> bash:
```
sudo apt install apache2-dev -y
```

## Setup trucknet
(if you'll be using sevenandahalf underneath that site, which I always will)

Follow brief instructions at: <https://github.com/charliemacquarie/trucknet>

## Initial setup for sevenandahalf
Create a virtual environment in which to install the app. Do this inside /usr/local/www/, creating the www directory if it does not exist already. Note that you may have to `sudo` to perform this command.
> bash:
```
python3 -m venv venv
```

Make your user the owner of the virtual environment to allow you to perform the later commands.
> bash:
```
sudo chown $USER:$USER -R venv/
```

Activate the environment you just created.
> bash:
```
source venv/bin/activate
```

Install sevenandahalf into the virtual environment:
> bash:
```
pip install https://charliemacquarie.com/software/sevenandahalf/dist/sevenandahalf-1.2.1-py3-none-any.whl
```

## Install and configure mod_wsgi
mod_wsgi is very configurable, and complete instructions are found in their docs:
- to install mod_wsgi: https://modwsgi.readthedocs.io/en/master/user-guides/quick-installation-guide.html
- to configure mod_wsgi: https://modwsgi.readthedocs.io/en/master/user-guides/quick-configuration-guide.html
- to note using virtual environments with mod_wsgi (necessary for a flask app): https://modwsgi.readthedocs.io/en/master/user-guides/virtual-environments.html

Note that you should still have the virtual environment activated while installing and configuring mod_wsgi (and for the rest of these instructions)

Download and unpack the source code. (Note that the numbers in the release file may change depending on which release you are seeking.)
> bash:
```
wget https://github.com/GrahamDumpleton/mod_wsgi/archive/refs/tags/4.9.0.tar.gz
tar xvfz 4.9.0.tar.gz
```

Change into the directory for the source code and run the configuration script.
> bash:
```
cd mod_wsgi-4.9.0/
./configure
```

Perform the installation
> bash:
```
make
sudo make install
```

### Activate mod_wsgi
You have to create the correct load file in the apache2 configurations, for Rasberrypi this is usually at /etc/apache2/mods-available/. The file should be called wsgi.load, so:
> bash:
```
sudo nano /etc/apache2/mods-available/wsgi.load
```
and add the single line `LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so` to the file.

Then the module must be activated with:
> bash:
```
sudo a2enmod wsgi
```

Finally, apache2 should be restarted for the changes to take effect.
> bash:
```
systemctl restart apache2
```

At this point it may be advisable to check the apache error log (usually at /var/log/apache2/error.log) to verify things are working correctly. You're looking for a line resembling something like: `[Tue Mar 29 21:47:32.679604 2022] [mpm_event:notice] [pid 28297:tid 1996198336] AH00489: Apache/2.4.52 (Raspbian) mod_wsgi/4.9.0 Python/3.9 configured -- resuming normal operations` if things are going smoothly.

### Setup mod_ssl
Since sevenandahalf accesses the user's location, it only works over https://. This is a good point to enable the ssl module, which serves the site over https.
> bash:
```
sudo a2enmod ssl
sudo a2enmod rewrite
sudo a2ensite default-ssl
```
All the further site configurations will be placed into the default-ssl.conf file at `/etc/apache2/sites-enabled/default-ssl.conf`

Open the ssl site conf file:
> bash:
```
sudo nano /etc/apache2/sites-enabled/default-ssl.conf
```

And add (or uncomment -- you may find them already in there) the following lines:
> apache2 config text:
```
SSLCertificateFile      /etc/ssl/certs/ssl-cert-snakeoil.pem
SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
```

_note that there is a complete sample default-ssl.conf text at the bottom of this document_

(restart for the changes to take effect, check the logs if you want to make sure it's working)
> bash:
```
systemctl restart apache2
```

**NOTE** as may be clear, this site uses a self-signed ssl certificate, which will throw the user a huge giant security warning that they'll have to click around the first time they access the site. There is no way around this, because sevenandahalf is designed to work on a local network that's not connected to the internet, so signed certificates will never work because there may not be an internet connection by which to verify them. It's probably a problem to encourage the user to click around such a warning, but whatever, this app has a highly specific group of people who will ever use it. You can explain to them the deal.

### Configure the site to use mod_wsgi and the wsgi script
Your default site configuration needs to be edited to add the Alias for the wsgi script (where the app will be imported) and give apache access to read the files in this location. You may also want to enable DaemonProcess mode which allows changes to the source code to take effect without restarting the server. More on this can be found in the [mod_wsgi documentation](https://modwsgi.readthedocs.io/en/master/user-guides/quick-configuration-guide.html#delegation-to-daemon-process).

Open the site configuration file for editing.
> bash:
```
sudo nano /etc/apache2/sites-enabled/default-ssl.conf
```

Add in the following configuration lines:
> apache2 config text:
```
WSGIDaemonProcess trucknet.freq processes=2 threads=15 display-name=%{GROUP} python-home=/usr/local/www/venv
WSGIProcessGroup trucknet.freq

WSGIScriptAlias /where /usr/local/www/sevenandahalf/where.wsgi

<Directory /usr/local/www/sevenandahalf>
<IfVersion < 2.4>
        Order allow,deny
        Allow from all
</IfVersion>
<IfVersion >= 2.4>
        Require all granted
</IfVersion>
</Directory>
```

_note that there is a complete sample default-ssl.conf text at the bottom of this document_

Create the wsgi script the site will use to serve sevenandahalf (and create the /usr/local/www/sevenandahalf directory if it doesn't exist already).
> bash:
```
sudo nano /usr/local/www/sevenandahalf/where.wsgi
```
and inside this file add the following lines:
> python/wsgi:
```
import sevenandahalf

application = sevenandahalf.create_app()
```

(restart for the changes to take effect, check the logs if you want to make sure it's working)
> bash:
```
systemctl restart apache2
```

## Setup sevenandahalf!
Tell the system what/where the flask app is to use the setup processes.
> bash:
```
export FLASK_APP=sevenandahalf
```

Go get a list of all the USGS topographic maps to use to initialize your sevenandahalf
> bash:
```
wget https://charliemacquarie.com/librarystorage/resources/topomaps_all.zip
unzip topomaps_all.zip
```

the resulting csv file `topomaps_all.csv` will be the file you should use with the get-maps command.

Make your user the owner of the document root so that the necessary folders can be created by the script.
> bash:
```
sudo chown -R $USER:$USER /var/www/html/
```

Download some maps with the get-maps command. (Note that depending on what you decide to download, this command may take a *very* long time to run and download all the maps you've selected.)
> bash:
```
flask get-maps --web-root /var/www/html/ topomaps_all.csv
```

Initialize the database for the maps you downloaded
> bash:
```
flask init-db
```

Your site should now be ready to visit! https://192.168.1.153 in a browser on my network, yours may vary.

### Setup the secret key in the config file

Follow directions at (https://flask.palletsprojects.com/en/2.1.x/tutorial/deploy/#configure-the-secret-key)

## Full examples of configs
### default-ssl.conf
Example of entire contents of config file:
```
<IfModule mod_ssl.c>
	<VirtualHost _default_:443>
		ServerName www.trucknet.freq
		ServerAlias trucknet.freq
		ServerAdmin webmaster@trucknet.freq

		DocumentRoot /var/www/html

		LogLevel info

		ErrorLog ${APACHE_LOG_DIR}/error.log
		CustomLog ${APACHE_LOG_DIR}/access.log combined

		SSLEngine on

		SSLCertificateFile	/etc/ssl/certs/ssl-cert-snakeoil.pem
		SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

		<FilesMatch "\.(cgi|shtml|phtml|php)$">
				SSLOptions +StdEnvVars
		</FilesMatch>
		<Directory /usr/lib/cgi-bin>
				SSLOptions +StdEnvVars
		</Directory>

		WSGIDaemonProcess trucknet.freq processes=2 threads=15 display-name=%{GROUP} python-home=/usr/local/www/venv
		WSGIProcessGroup trucknet.freq

		WSGIScriptAlias /where /usr/local/www/sevenandahalf/where.wsgi

		<Directory /usr/local/www/sevenandahalf>
		<IfVersion < 2.4>
			Order allow,deny
			Allow from all
		</IfVersion>
		<IfVersion >= 2.4>
			Require all granted
		</IfVersion>
		</Directory>

	</VirtualHost>
</IfModule>
```
