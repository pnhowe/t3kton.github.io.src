Install From Prebuilt Packages
==============================

Building Packages
-----------------

NOTE: To build the packages, you will need a temporary Ubuntu Xenial install, you can
use the target contractor VM if you don't mind a little extra stuff laying arround.

install the required build tools, the PPA has a few required packages for building
and installing::

  add-apt-repository ppa:pnhowe/t3kton
  apt update
  apt install -y git respkg build-essential dpkg-dev debhelper python3-dev python3-setuptools nodejs npm nodejs-legacy liblzma-dev

Create an empty directory, and cd into it

first clone the contractor and related projects::

  git clone https://github.com/T3kton/contractor.git
  git clone https://github.com/T3kton/contractor_plugins.git
  git clone https://github.com/T3kton/subcontractor.git
  git clone https://github.com/T3kton/subcontractor_plugins.git
  git clone https://github.com/T3kton/resources.git

now to build Contractor, first we need to get the node requirements for the UI, and fix a bug with react-toolbox::

  cd contractor
  cd ui && npm install && cd ..
  sed s/"export Ripple from '.\/ripple';"/"export { default as Ripple } from '.\/ripple';"/ -i ui/node_modules/react-toolbox/components/index.js
  sed s/"export Tooltip from '.\/tooltip';"/"export { default as Tooltip } from '.\/tooltip';"/ -i ui/node_modules/react-toolbox/components/index.js

now build the packages::

  make dpkg
  make respkg
  mv *.respkg ..
  cd ..

now to build the others::

  for i in subcontractor contractor_plugins subcontractor_plugins; do cd $i && make dpkg && cd ..; done

and build the resources, the make in the resources can take a while, you may want
to add `-j4` (replace the 4 with the number of compile process to parralaize)::

  cd contractor_plugins
  make respkg
  mv *.respkg ..
  cd ../resources
  make respkg
  mv *.respkg ..
  cd ..

copy the .deb and .respkg files from the build server to the target contractor vm.

on the target contractor vm setup repos and install some required tools::

  add-apt-repository ppa:pnhowe/t3kton
  apt update
  apt install -y bind9 postgresql-9.5

Install Packages::

  dpkg -i contractor_*.deb contractor-plugins_*.deb
  apt install -f

Installing from pre-built packages
----------------------------------

Setup the PPA which has the built packages::

  add-apt-repository ppa:pnhowe/t3kton
  apt update
  apt install -y respkg contractor contractor_plugins subcontractor subcontractor_plugins
