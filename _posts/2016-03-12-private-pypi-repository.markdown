---
title:  "Private PyPI Repository"
date:   2013-12-03 19:2:23
categories: [python]
tags: [python]
---
Recently Ive been doing some Python development on a network without public internet access. Our team had been bringing in common python libraries and placing them in a shared network location, but realized that it would be handy to host our own internal PyPi server for use of pip during development/deployment.

After investigating a number of different options available I decided to settle on DjangoPyPi2 as the PyPi server.  It allowed for an entirely self-contained configuration of both the PyPi server and all hosted packages.  DjangoPyPi2 provides a nice web application front end for package search and management. The installation and configuration steps described below are for a start to finish configuration of DjangoPyPi2 hosted using Gunicorn and Supervisor.  I decided to write this post because there were a few problems I had to solve along the way that I thought could make this process easier for others.


## Environment Configuration

To start off we need a way to simulate an install on a system without internet access.  I did this using an CentOS 6.4 64bit image inside of Oracle VirtualBox, both freely available online.  I created this image using the minimal installation in the CentOS installation wizard.  Once the image is created we will need to get all required packages onto the image.

We could accomplish this by allowing the VM to temporarily connect to the internet and download the packages, but since we want to simulate a fully disconnected install, I chose not to do this.  First, let us set up the VM in VirtualBox to only allow communication between our guest and the host machine.  This is done by selecting our VM in the Oracle VM Virtual Box Manager and opening its settings.  Under the Network tab select Adapter 1 and set it to Host-only Adapter.  After accepting this change, you will need to ensure the VM is configured to communicate externally.  In the CentOS VM terminal enter the following:

```
ifconfig
```

If the result only contains a lo adapter you will need to configure the eth0 adapter.  This can be done as follows by editing /etc/sysconfig/network-scripts/ifscfg-eth0 to be configured as follows:

```
NM_CONTROLLED="no"
ONBOOT="yes"
```

Now all that should be required is to restart the network service and verify eth0 adapter is available and configured with an IP:

```
service network restart
ifconfig
```
You should see an output similar to the image below:

## Software Installation

Now we need to load all the required packages onto the VM.  Firstly, download the following tar.gz packages from pypi.python.org to your local machine, all latest except for Django:

Django (1.5.1)
djangopypi2
django-registration
docutils
gunicorn
meld3
pip
pkginfo
setuptools
setuptools-git
supervisor
virtualenv

You can also accomplish this if you have pip on your local machine using the following command:

```
pip install --download /local/package/path virtualenv supervisor gunicorn setuptools-git djangopypi2 pip
```

All of the files should then be SCPed onto the CentOS VM.  I used MobaXterm as it includes an integrated SCP GUI tool once youve established an SSH session.  I placed them under /tmp on my VM.

## Python Tools Install

To perform the install using pip and virtualenv we need to install them into the systems python install.  This can be done by executing the following commands from where the packages were copied to on your VM (/tmp for me):

tar xvf setuptools-1.4.1.tar.gz
tar xvf pip-1.4.1.tar.gz
tar xvf virtualenv-1.10.1.tar.gz
cd setuptools-1.4.1
python setup.py install
cd ../pip-1.4.1
python setup.py install
cd ../virtualenv-1.10.1
python setup.py install

Now we can get to the business of actually installing PyPi, Gunicorn and dependencies into a virtualenv:

cd /opt
# Setup VirtualEnv and install DjangoPyPi2 into it
virtualenv djangopypi2
source djangopypi2/bin/activate
pip install --no-index -f /tmp setuptools-git
pip install --no-index -f /tmp gunicorn djangopypi2
# Configure DjangoPyPi2 to use /opt/djangopypi2 as its resource location
DJANGOPYPI2_ROOT=/opt/djangopypi2 manage-pypi-site syncdb
DJANGOPYPI2_ROOT=/opt/djangopypi2 manage-pypi-site collectstatic
DJANGOPYPI2_ROOT=/opt/djangopypi2 manage-pypi-site loaddata initial
# Exit VirtualEnv
deactivate

## Supervisor Install

Supervisor can either be configured manually using the source install or retrieved using whatever package management system available on your distribution of Linux.  This would be recommended, but since I wanted to demonstrate a completely disconnected install I chose to install form source.  This will mean that supervisord will not be configured to start on reboot.  There is a good resource of init scripts at https://github.com/Supervisor/initscripts that could be referenced to augment this installation.

Lets get to the supervisor configuration:

pip install --no-index -f /tmp supervisor
# Create boilerplate supervisord configuration file
echo_supervisord_conf > /etc/supervisord.conf

Edit the /etc/supervisord.conf file to contain the following section:

[program:djangopypi2]
directory=/opt/djangopypi2
command=/opt/djangopypi2/bin/gunicorn_django djangopypi2.website.settings -b 0.0.0.0:80
environment=PATH="/opt/djangopypi2/bin",DJANGOPYPI2_ROOT="/opt/djangopypi2"
autostart=true
autorestart=true
redirect_stderr=true

All that is left now is to start it up and test it on the host machine:

supervisord
# temporarily disable firewall so port 80 can be accessed externally
service iptables stop
# identify ip address to access in browser
ifconfig

Browse to the IP address seen in the ifconfig output (inet addr ) in your favorite browser and you should see the DjangoPyPi2 web page.

If the web server does not seem to be running you can diagnose supervisor and its child processes by running supervisorctl at the command line.  You will also want to configure your firewall to allow communication on port 80 or whatever port you choose for this service.

Jekyll also offers powerful support for code snippets:

``` ruby
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
```

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
