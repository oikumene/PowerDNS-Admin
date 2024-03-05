On [FreeBSD](https://www.freebsd.org/), most software is installed using `pkg`. You can always build from source with the Ports system. This method uses as many binary ports as possible, and builds some python packages from source. It installs all the required runtimes in the global system (e.g., python, node, yarn) and then builds a virtual python environment in `/opt/python`. Likewise, it installs powerdns-admin in `/opt/powerdns-admin`.

### Build an area to host files

```sh
mkdir -p /opt/python
```

### Install prerequisite runtimes: python, node, yarn

```sh
sudo pkg install git python3 curl node21 yarn-node21
sudo pkg install libxml2 libxslt pkgconf py39-xmlsec
sudo pkg install mysql81-client openldap26-client
```

Note: package of MySQL client may be mysql80-client if you want.

## Check Out Source Code
_**Note:**_ Please adjust `/opt/powerdns-admin` to your local web application directory

```sh
git clone https://github.com/PowerDNS-Admin/PowerDNS-Admin.git /opt/powerdns-admin
cd /opt/powerdns-admin
```

## Make Virtual Python Environment

Make a virtual environment for python. Activate your python3 environment and install libraries. It's easier to install some python libraries as system packages, so we add the `--system-site-packages` option to pull those in.

-> Note: xmlsec 1.3.13 does not support xmlsec1 >= 1.3. When it supports xmlsec1 > 1.3, you may build it yourself and remove `--system-site-packages` option. You will need to install `xmlsec1` in such case.

```sh
python3 -m venv /web/python --system-site-packages
source /web/python/bin/activate
/web/python/bin/python3 -m pip install --upgrade pip wheel
# the -C arg options are needed to build python-ldap
pip3 install -C --build-option=build_ext -C --build-option=--include-dirs=/usr/local/include -C --build-option=--library-dirs=/usr/local/lib -r requirements.txt
```

## Configuring PowerDNS-Admin

NOTE: The default config file is located at `./powerdnsadmin/default_config.py`. If you want to load another one, please set the `FLASK_CONF` environment variable. E.g.
```sh
cp configs/development.py /opt/powerdns-admin/production.py
export FLASK_CONF=/opt/powerdns-admin/production.py
```

### Update the Flask config

Edit your flask python configuration. Insert values for the database server, user name, password, etc.

```sh
vim $FLASK_CONF
```

Edit the values below to something sensible
```python
### BASIC APP CONFIG
SALT = '[something]'
SECRET_KEY = '[something]'
BIND_ADDRESS = '0.0.0.0'
PORT = 9191
OFFLINE_MODE = False

### DATABASE CONFIG
SQLA_DB_USER = 'pda'
SQLA_DB_PASSWORD = 'changeme'
SQLA_DB_HOST = '127.0.0.1'
SQLA_DB_NAME = 'pda'
SQLALCHEMY_TRACK_MODIFICATIONS = True
```

Be sure to uncomment one of the lines like `SQLALCHEMY_DATABASE_URI`.

### Initialise the database

```sh
export FLASK_APP=powerdnsadmin/__init__.py
flask db upgrade
```

### Build web assets

```sh
yarn install --pure-lockfile
flask assets build
```

## Running PowerDNS-Admin

Now you can run PowerDNS-Admin by command

```sh
./run.py
```

Open your web browser and go to `http://localhost:9191` to visit PowerDNS-Admin web interface. Register a user. The first user will be in the Administrator role.

### Running at startup

This is good for testing, but for production usage, you should use gunicorn or uwsgi. See [Running PowerDNS Admin with Systemd, Gunicorn and Nginx](../web-server/Running-PowerDNS-Admin-with-Systemd-Gunicorn-and-Nginx.md) for instructions.

Create a startup script `pdnsadmin` in `/usr/local/etc/rc.d`.

```
#!/bin/sh

# PROVIDE: pdnsadmin
# REQUIRE: DAEMON pdns
# KEYWORD: shutdown

#
# Add the following line to /etc/rc.conf to enable pdnsadmin:
# pdnsadmin_enable (bool):      Set to "NO" by default.
#                               Set it to "YES" to enable pdnsadmin.
#

. /etc/rc.subr

name=pdnsadmin
desc="PowerDNS Admin web UI daemon"
rcvar=${name}_enable

load_rc_config ${name}

workdir="/opt/web/powerdns-admin
command="/web/python/bin/gunicorn"
pidfile="/var/run/powerdns-admin.pid"
sockdir="/var/run/powerdns-admin"
socket="${sockdir}/powerdns-admin.sock"

start_cmd=start_cmd
stop_cmd='pkill -TERM -U pdns -F ${pidfile}; sleep 3'

pdnsadmin_enable=${pdnsadmin_enable:-NO}

start_cmd()
{
        check_startmsgs && echo "Starting ${name}."
        if [ !-d ${sockdir} ]; then
            mkdir -p ${sockdir}
            chown pdns:pdns ${sockdir}
        fi
        cd ${workdir} && daemon -f -u pdns -p ${pidfile} ${command} --bind unix:${socket} --log-syslog 'powerdnsadmin:create_app()'
}

run_rc_command $1
```

And enable it through `/etc/rc.conf`.

```
pdnsadmin_enable="YES"
```

Now, you may start PowerDNS Admin with `service pdnsadmin start`.
