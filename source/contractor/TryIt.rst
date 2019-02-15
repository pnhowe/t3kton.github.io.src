Give it a Try
=============

*Warning*: Contractor is made to solve complex problems, for the most part it presents
a must less complex interface to you.  That being said, there is a bit of a hurdle
to get it up and running, depending on what you are trying to automate.  So, buckle
in, this is going to be a lot of fun.

This demo will result in a VM in a private /24 network that can create and destroy
other VMs in that network.  You can use either VCenter or VirtualBox.  After the
Contractor VM is up and running, you can install more blueprints and plugins to do
docker or other foundations.  The requirement for the private network comes primarally from
the setupWizzard's default configuration, if you feel comfortable with modifying
the setupWizzard and the Apache Configurations, you can use blueprints and plugins
for hosted providers such as AWS and Azure.

NOTE: setupWizzard is going to re-write some bind config files, so don't edit them
until after the install is complete.

Installing
----------

Create an Ubuntu Xenial VM, name it `contractor`, set the fqdn to `contractor.site1.local`
Ideally it should be in a /24 network.  Ip addresses offsets from 2 to 20 will be
set as reserved (so that contractor will not auto asign them) and offsets 21 - 29
will be used for a dynamic DHCP pool. Offset 1 is assumed to te be the gateway.
All these values can be adjusted either in the setupWizzard file before it is run,
or after it setup, you can use the API/UI to edit these values.
The DNS setver will be set for the contractor VM, and bind on the contractor vm will
be set to forward to the DNS server that was origionally configured on the VM.

The default when installing from package is to use postgres for the database.
sqlite is also supported, and the prefered way for developmnet.  You can edit the
`/etc/contractor/master.conf` just before the `manage.py migrate` command to
enable the sqlite option if desired, make sure to skip the postgres setup.  Keep
in mind that sqlite option requires making the database writeable by `www-data.www-data`.

From Source
~~~~~~~~~~~

  :doc:`TryIt_source`


From Packages
~~~~~~~~~~~~~

  :doc:`TryIt_packaged`


Setup
-----

Install Postgres::

  apt install -y postgresql-client postgresql-9.5

Create the postgres db::

  su postgres -c "echo \"CREATE ROLE contractor WITH PASSWORD 'contractor' NOSUPERUSER NOCREATEDB NOCREATEROLE LOGIN;\" | psql"
  su postgres -c "createdb -O contractor contractor"

the ubuntu toml package is to old::

  apt install -y python3-pip
  pip3 install toml --upgrade

Now to create the db::

  /usr/lib/contractor/util/manage.py migrate

Install base os config::

  respkg -i contractor-os-base_0.1.respkg
  respkg -i contractor-ubuntu-base_0.1.respkg

Now to enable plugins.
We use manual for misc stuff that is either pre-configured or handled by something else::

  respkg -i contractor-plugins-manual_0.1.respkg

if you are using vcenter::

  respkg -i contractor-plugins-vcenter_0.1.respkg

if you are using virtualbox::

  respkg -i contractor-plugins-virtualbox_0.1.respkg

do manual plugin again so it can cross link to the other plugins::

  respkg -i contractor-plugins-manual_0.1.respkg

Now to setup some base info, and configure bind::

  /usr/lib/contractor/setup/setupWizzard

Restart bind with new zones::

  systemctl bind9 restart

This VM needs to use the contractor generated dns, so edit
`/etc/network/interfaces` to set the dns server to 127.0.0.1
then, reload networking configuration::

  systemctl restart networking

Now to disable the extra apache site::

  a2dissite 000-default
  systemctl restart apache2

you might want to tweek `/etc/apache2/sites-available/contractor.conf`

now take a look at the contractor ui at http://<contractor ip>

now we will setup subcontractor::

  apt install tftpd-hpa
  respkg -i contractor-ipxe_0.1.respkg

now edit /etc/subcontractor.conf
enable the modules you want to use, remove the ';' and set the 0 to a 1.
The 1 means one task for that plugin at a time, if you want things to go faster,
you can try 2 or 4.  Depending on the plugin, the resources of your vm, etc.

edit `/etc/subcontractor.conf` in the dhcpd section, make sure interface and tftp_server
are correct, tftp_server should be the ip of the vm

now start up subcontractor::

  systemctl start subcontractor
  systemctl start dhcpd

make sure it's running::

  systemctl status subcontractor
  systemctl status dhcpd

optional, edit `/etc/default/tftpd-hpa` and add '-v ' to TFTP_OPTIONS.  This will
cause tfptd to log transfers to syslog.  This can be helpfull in troubleshooting
boot problems. Make sure to run `systemctl restart tftpd-hpa` to reload.

to service static resources (such as the OS installers) you will need to setup
a static web server.  First create the directory::

  mkdir -p /var/www/static

now create `/etc/apache2/sites-available/static.conf` with the following::

  <VirtualHost *:80>
    ServerName static
    ServerAlias static.<domain>

    DocumentRoot /var/www/static

    LogFormat "%a %t %D \"%r\" %>s %I %O \"%{Referer}i\" \"%{User-Agent}i\" %X" static_log
    ErrorLog ${APACHE_LOG_DIR}/static_error.log
    CustomLog ${APACHE_LOG_DIR}/static_access.log static_log
  </VirtualHost>

now enable the site::

  a2ensite static
  systemctl restart apache2
