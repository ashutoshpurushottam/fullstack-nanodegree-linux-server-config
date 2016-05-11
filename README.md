# Project 5 of the Udacity Full Stack Nanodegree programme
_Configuring a Linux server to host a web app securely._

# Server details
IP address: ~~`52.11.206.40`~~

SSH port: `2200`

URL: ~~`http://ec2-52-11-206-40.us-west-2.compute.amazonaws.com`~~

_Note that since I have now graduated, the original server has been disabled. You might find
something else there now, but what follows would apply when setting up a new server._

# Configuration changes
## Add user
Add user `grader` with command: `useradd -m -s /bin/bash grader`

## Add user grader to sudo group
Assuming your Linux distro has a `sudo` group (like Ubuntu 16.04), simply add the user to
this admin group:
```
usermod -aG sudo grader
```

## Update all currently installed packages

`apt-get update` - to update the package indexes

`apt-get upgrade` - to actually upgrade the installed packages

If at login the message `*** System restart required ***` is display, run the following
command to reboot the machine:

`reboot`

## Set-up SSH keys for user grader
As root user do:
```
mkdir /home/grader/.ssh
chown grader:grader /home/grader/.ssh
chmod 700 /home/grader/.ssh
cp /root/.ssh/authorized_keys /home/grader/.ssh/
chown grader:grader /home/grader/.ssh/authorized_keys
chmod 644 /home/grader/.ssh/authorized_keys
```

Can now login as the `grader` user using the command:
`ssh -i ~/.ssh/udacity_key.rsa grader@52.11.206.40`

## Fix sudo resolve host error
When the `grader` user issues a `sudo` command, got the following warning:
`sudo: unable to resolve host ip-10-20-47-177`

To fix this, the hostname was added to the loopback address in the `/etc/hosts` file
so that th first line now reads:
`127.0.0.1 localhost ip-10-20-47-177`

This solution was found on the Ubuntu Forms [here][1].

## Disable root login
Change the following line in the file `/etc/ssh/sshd_config`:

From `PermitRootLogin without-password` to `PermitRootLogin no`.

Also, uncomment the following line so it reads:
```
PasswordAuthentication no
```

Do `service ssh restart` for the changes to take effect.

Will now do all commands using the `grader` user, using `sudo` when required.

## Change timezone to UTC
Check the timezone with the `date` command. This will display the current timezone after the time.
If it's not UTC change it like this:

`sudo timedatectl set-timezone UTC`

## Change SSH port from 22 to 2200
Edit the file `/etc/ssh/sshd_config` and change the line `Port 22` to:

`Port 2200`

Then restart the SSH service:

`sudo service ssh restart`

Will now need to use the following command to login to the server:

`ssh -i ~/.ssh/udacity_key.rsa grader@52.11.206.40 -p 2200`

## Configuration Uncomplicated Firewall (UFW)
By default, block all incoming connections on all ports:

`sudo ufw default deny incoming`

Allow outgoing connection on all ports:

`sudo ufw default allow outgoing`

Allow incoming connection for SSH on port 2200:

`sudo ufw allow 2200/tcp`

Allow incoming connections for HTTP on port 80:

`sudo ufw allow www`

Allow incoming connection for NTP on port 123:

`sudo ufw allow ntp`

To check the rules that have been added before enabling the firewall use:

`sudo ufw show added`

To enable the firewall, use:

`sudo ufw enable`

To check the status of the firewall, use:

`sudo ufw status`

## Install Apache to serve a Python mod_wsgi application
Install Apache:

`sudo apt-get install apache2`

Install the `libapache2-mod-wsgi` package:

`sudo apt-get install libapache2-mod-wsgi`

## Install and configure PostgreSQL
Install PostgreSQL with:

`sudo apt-get install postgresql postgresql-contrib`

To ensure that remote connections to PostgreSQL are not allowed, I checked
that the configuration file `/etc/postgresql/9.3/main/pg_hba.conf` only
allowed connections from the local host addresses `127.0.0.1` for IPv4
and `::1` for IPv6.

Create a PostgreSQL user called `catalog` with:

`sudo -u postgres createuser -P catalog`

You are prompted for a password. This creates a normal user that can't create
databases, roles (users).

Create an empty database called `catalog` with:

`sudo -u postgres createdb -O catalog catalog`

The Ubuntu documentation page on [PostgreSQL][3] was helpful.

## Install Flask, SQLAlchemy, etc
Issue the following commands:
```
sudo apt-get install python-psycopg2 python-flask
sudo apt-get install python-sqlalchemy python-pip
sudo pip install oauth2client
sudo pip install requests
sudo pip install httplib2
sudo pip install flask-seasurf
```

An alternative to installing system-wide python modules is to create a virtual
environment for each application using the [virualenv][4] package.

## Install Git version control software
`sudo apt-get install git`

## Clone the repository that contains Project 3 Catalog app
Move to the `/srv` directory and clone the repository as the `www-data` user.
The `www-data` user will be used to run the catalog app.
```
cd /srv
sudo mkdir fullstack-nanodegree-vm
sudo chown www-data:www-data fullstack-nanodegree-vm/
sudo -u www-data git clone https://github.com/SteveWooding/fullstack-nanodegree-vm.git fullstack-nanodegree-vm
```

## Update catalog.wsgi file for this installation
Absolute paths are updated to where the catalog is located. The application `secret_key` is set to
something random and the PostgreSQL for the `catalog` user is set. In this case, the file
looks like this, except the password and secret is not shown for security reasons.

```python
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/srv/fullstack-nanodegree-vm/vagrant/catalog')

from catalog import app as application
from catalog.database_setup import create_db
from catalog.populate_database import populate_database

application.secret_key = 'SECRET'  # This needs changing in production env

application.config['DATABASE_URL'] = 'postgresql://catalog:PASSWORD@localhost/catalog'
application.config['UPLOAD_FOLDER'] = '/srv/fullstack-nanodegree-vm/vagrant/catalog/item_images'
application.config['OAUTH_SECRETS_LOCATION'] = '/srv/fullstack-nanodegree-vm/vagrant/catalog/'
application.config['ALLOWED_EXTENSIONS'] = set(['jpg', 'jpeg', 'png', 'gif'])
application.config['MAX_CONTENT_LENGTH'] = 4 * 1024 * 1024  # 4 MB

# Create database and populate it, if not already done so.
create_db(application.config['DATABASE_URL'])
populate_database()
```

## Update the Google OAuth client secrets file
Fill in the `client_id` and `client_secret` fields in the file `g_client_secrets.json`.
Also change the `javascript_origins` field to the IP address and AWS assigned URL of the host.
In this instance that would be:
`"javascript_origins":["http://52.11.206.40", "http://ec2-52-11-206-40.us-west-2.compute.amazonaws.com"]`

These addresses also need to be entered into the Google Developers Console -> API Manager
-> Credentials, in the web client under "Authorized JavaScript origins".

## Update the Facebook OAuth client secrets file
In the file `fb_client_secrets.json`, fill in the `app_id` and `app_secret` fields with
the correct values.

In the Facebook developers website, on the Settings page, the website URL needs to read
`http://ec2-52-11-206-40.us-west-2.compute.amazonaws.com`. Then in the "Advanced" tab,
in the "Client OAuth Settings" section, add `http://ec2-52-11-206-40.us-west-2.compute.amazonaws.com`
and `http://52.11.206.40` to the "Valid OAuth redirect URIs" field. Then save these changes.


## Configure Apache2 to serve the app
To serve the catalog app using the Apache web server, a virtual host configuration file
needs to be created in the directory `/etc/apache2/sites-available/`, in this case called
`catalog-app.conf`. Here are its contents:

```
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost

        # Define WSGI parameters. The daemon process runs as the www-data user.
        WSGIDaemonProcess catalog user=www-data group=www-data threads=5
        WSGIProcessGroup catalog
        WSGIApplicationGroup %{GLOBAL}

        # Define the location of the app's WSGI file
        WSGIScriptAlias / /srv/fullstack-nanodegree-vm/vagrant/catalog/catalog.wsgi

        # Allow Apache to serve the WSGI app from the catalog app directory
        <Directory /srv/fullstack-nanodegree-vm/vagrant/catalog/>
                Require all granted
        </Directory>

        # Setup the static directory (contains CSS, Javascript, etc.)
        Alias /static /srv/fullstack-nanodegree-vm/vagrant/catalog/catalog/static

        # Allow Apache to serve the files from the static directory
        <Directory  /srv/fullstack-nanodegree-vm/vagrant/catalog/catalog/static/>
                Require all granted
        </Directory>

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Documentation for the WSGI parameters can be found [here][5].

Disable the default virtual host with:

`sudo a2dissite 000-default.conf`

Then enable the catalog app virtual host:

`sudo a2ensite catalog-app.conf`

To make these Apache2 configuration changes live, reload Apache:

`sudo service apache reload`

The catalog app should now be available at `http://52.11.206.40` and
`http://ec2-52-11-206-40.us-west-2.compute.amazonaws.com`

[This][6] was a useful guide to setting up a Flask app on Apache, though the `Directory`
permissions in the virtual host file were out of date (now it's `Require all granted`
to allow all clients to access the server).

## NTP Daemon
The NTP daemon was installed to continuously adjust the system clock so no large differences
in time develop. Install with:

`sudo apt-get install ntp`

The configuration file `/etc/ntp.conf` was left in its default state. Time servers may be
added or changed via this file.

Did a check of the offsets using the command `sudo ntpq -p` and got the result:
```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 deekayen.net    209.51.161.238   2 u   55   64    1   77.251  -1256.9   0.000
 clock.team-cymr 204.9.54.119     2 u   54   64    1   56.417  -1256.3   0.000
 time-b.timefreq .ACTS.           1 u   53   64    1   33.347  -1258.6   0.000
 natasha.netwurx 209.242.224.11   2 u   52   64    1   51.188  -1258.7   0.000
 juniperberry.ca 140.203.204.77   2 u   51   64    1  139.969  -1256.5   0.000
```

Notice the quite large offset from the real time of over a second. A while later,
these offsets were dramatically reduced (figures are given in milliseconds).
```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 deekayen.net    209.51.161.238   2 u   10   64    7   77.254    2.153   0.272
 clock.team-cymr 204.9.54.119     2 u   16   64    7   56.434    2.759   0.710
 time-b.timefreq .ACTS.           1 u    8   64    7   33.405    0.545   0.136
 natasha.netwurx 209.242.224.11   2 u   13   64    7   51.342    0.214   0.358
 juniperberry.ca 131.188.3.220    2 u   16   64    7  140.157    2.602   0.155
```

This means the NTP daemon is working correctly.

# Extra Credit Configurations
## Automatic Updates
Automatic security updates have been implementing using the instructions in the
Ubuntu documentation, [Automatic Updates][7].

Check that the `unattended-upgrades` package is installed:

`sudo apt-get install unattended-upgrades`

Check the configuration file `/etc/apt/apt.conf.d/50unattended-upgrades` to see which class
of updates get installed. It was left at the default of security updates only.

Change the configuration file `/etc/apt/apt.conf.d/10periodic` to specify how often updates
and other tasks should be performed. It's contents are now:
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```
So the package list is updated daily and security updates will be downloaded and installed daily.
Every week the local download archive will be cleaned.

Unattended package installation can be monitiored by reviewing the `apt` histor log file located
here: `/var/log/apt/history.log`.

## Monitor for Repeated Failed Login Attempts
The program Fail2Ban will be used to block IP addresses from which unsuccessful login attempts
have occurred. This is on top of using a non-standard SSH port number and requiring an RSA key
to login. The following guide was useful in this task: [How To Protect SSH with Fail2Ban on Ubuntu 14.04][8].
Also this guide for setting up Fail2ban when the UFW is being used: [UFW with Fail2ban – Quick Secure Setup][9].
Although I needed [this page][10] to confirm a detail in that post.

Install Fail2Ban:

`sudo apt-get install fail2ban`

Install Sendmail to send email notifications of banned IP addresses:

`sudo apt-get install sendmail`

Copy the Fail2Ban configuration file to a local config file that will override it. This way, if
their updates to the default config file, they will not have to be merged with any changes
the user makes.

`sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`

Edit this file with:

`sudo nano /etc/fail2ban/jail.local`

I set the following settings that differ from the default. Within the `[DEFAULT]` section:
```
bantime = 1800

destemail = [my email address]

action = %(action_mwl)s
```
This bans an IP address for 1800 seconds and sends an email to the address specified. The action
means an email notification will be sent with logging extract included.

The `ssh` section is as follows:
```
[ssh]

enabled  = true
banaction = ufw-ssh
port     = 2200
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
```
The `banaction` is set to run a custom action `ufw-ssh` which will be defined below. It uses
the UFW frontend to the iptables firewall. This will mean blocked IP address will appear
in the output of `sudo ufw status`. The port is set to our customised SSH port.

Contents of `/etc/fail2ban/action.d/ufw-ssh.conf` is:
```
[Definition]
actionstart =
actionstop =
actioncheck =
actionban = ufw insert 1 deny from <ip> to any port 2200
actionunban = ufw delete deny from <ip> to any port 2200
```
So Fail2Ban will insert a deny rule as the first rule from the relevant IP address for
the custom SSH port 2200. After the 1800 second ban time, this rule will be deleted.

## System Monitoring
[Glances][11] is a full-featured system monitor. It can also monitor processes. Install Glances with:

`sudo apt-get install glances`

To monitor Apache and Postgres, the following lines were added to the Glances config file
`/etc/glances/glances.conf`:
```
list_1_description=Apache Server
list_1_regex=.*apache.*
list_2_description=Postgres
list_2_regex=.*postgres.*
```
So now when `glances` is run, it will report the number of processes the are running for
Apache and Postgres. If one of these is not running, it will display `NOT RUNNING` next
to the name.

[1]: http://askubuntu.com/questions/59458/error-message-when-i-run-sudo-unable-to-resolve-host-none
[2]: http://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt
[3]: https://help.ubuntu.com/community/PostgreSQL
[4]: https://virtualenv.readthedocs.org/en/latest/
[5]: https://code.google.com/p/modwsgi/wiki/ConfigurationDirectives
[6]: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
[7]: https://help.ubuntu.com/lts/serverguide/automatic-updates.html
[8]: https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04
[9]: http://blog.productlaunchjourney.com/2014/08/ufw-with-fail2ban-quick-secure-setup.html
[10]: http://askubuntu.com/questions/54771/potential-ufw-and-fail2ban-conflicts
[11]: https://pypi.python.org/pypi/Glances
