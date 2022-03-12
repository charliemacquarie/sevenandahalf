# Seven And A Half

sevenandahalf is a web app that takes your location and shows you the USGS topographic maps that you are located on.

It also includes a script to read a .csv file of maps, specify which ones you want using a year cutoff, a US State list, or a bounding box, download the maps, and initialize the app database with that data to allow you to access the maps.

To run, this app requires an apache2 webserver configured to use the WSGI standard. Instructions and tips are included for mod_wsgi.

to install mod_wsgi: https://modwsgi.readthedocs.io/en/master/user-guides/quick-installation-guide.html

to configure mod_wsgi: https://modwsgi.readthedocs.io/en/master/user-guides/quick-configuration-guide.html

to note using virtual environment (necessary for a flask app): https://modwsgi.readthedocs.io/en/master/user-guides/virtual-environments.html

## Create a virtual environment in which to install the app:

```
python3 -m venv venv
```
_where to do this?_

## Activate the environment you just created:

```
source venv/bin/activate
```
## Install the app into the virtual environment:

```pip install https://charliemacquarie.com/software/sevenandahalf/dist/sevenandahalf-1.1.0-py3-none-any.whl
```
_check this will actually work_

## Tell the system what/where the flask app is to use the setup processes:

```
export FLASK_APP=sevenandahalf
```
