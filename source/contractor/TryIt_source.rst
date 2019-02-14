Install From Source
===================

Building requires running on a Ubuntu Xenial machine.

install the required build tools, the PPA has a few required packages for building
and installing::

  add-apt-repository ppa:pnhowe/t3kton
  apt update
  apt install -y git respkg build-essential python3-dev python3-setuptools nodejs npm nodejs-legacy liblzma-dev python3-django apache2 libapache2-mod-wsgi-py3 python3-werkzeug python3-psycopg2 python3-cinp python3-toml python3-jinja2

Create an empty directory, and cd into it

first clone the contractor and related projects::

  git clone https://github.com/T3kton/contractor.git
  git clone https://github.com/T3kton/contractor_plugins.git
  git clone https://github.com/T3kton/subcontractor.git
  git clone https://github.com/T3kton/subcontractor_plugins.git
  git clone https://github.com/T3kton/resources.git

now to build Contractor, first we need to get the node requirements for the UI, and fix a bug with react-toolbox::

  cd contractor
  cd ui && npm install
  sed s/"export Ripple from '.\/ripple';"/"export { default as Ripple } from '.\/ripple';"/ -i ui/node_modules/react-toolbox/components/index.js
  sed s/"export Tooltip from '.\/tooltip';"/"export { default as Tooltip } from '.\/tooltip';"/ -i ui/node_modules/react-toolbox/components/index.js
  cd ..

and build the resources, the make in the resources can take a while, you may want to replace the 2 of the `-j2` with the number of cores you are using::

  for i in contractor contractor_plugins resources; do cd $i && make -j2 respkg && mv *.respkg .. && cd ..; done

now to install the python, NOTE the Makefile will call './setup.py install' for you::

  cd contractor
  DESTDIR=/ make install
  cp debian/cron.d /etc/cron.d/contractor
  cp /etc/contractor/master.conf.sample /etc/contractor/master.conf
  ln -sf /etc/contractor/master.conf /usr/lib/python3/dist-packages/contractor/settings.py
  mkdir -p /etc/bind/contractor/zones
  mkdir -p /var/lib/contractor
  a2ensite contractor.conf
  a2enmod wsgi
  cd ..
  cd contractor_plugins
  DESTDIR=/ make install
  cd ..
  cd subcontractor
  DESTDIR=/ make install
  cp debian/dhcpd.service /lib/systemd/system/
  cp debian/subcontractor.service /lib/systemd/system/
  systemctl enable dhcpd
  systemctl enable subcontractor
  cd ..
  cd subcontractor_plugins
  DESTDIR=/ make install
  cd ..

  systemctl daemon-reload
  systemctl restart cron
