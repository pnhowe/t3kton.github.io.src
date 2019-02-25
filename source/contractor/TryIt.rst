Give it a Try
==============

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
Ideally it should be in a /24 network.  Offset 1 is assumed to be the gateway.
All these values can be adjusted either in the setupWizzard file before it is run,
or after it is setup, you can use the API/UI to edit these values.
The DNS server will be set for the contractor VM, and bind on the contractor vm will
be set to forward to the DNS server that was originally configured on the VM.

The default when installing from package is to use postgres for the database.
sqlite is also supported, and the prefered way for development.  You can edit the
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

Set up Infrastructure
~~~~~~~~~~~~~~~~~~~~~

To prevent your new contractor VM from taking over the network you are curently
connected to, you will need to configure a issloated network.  There are also
some other Infrastructure related tasks that need to be done.

:doc:`TryIt_vcenter_setup`

:doc:`TryIt_virtualbox_setup`

Install Requred Services
~~~~~~~~~~~~~~~~~~~~~~~~

Install Postgres::

  sudo apt install -y postgresql-client postgresql-9.5

Create the postgres db::

  sudo su postgres -c "echo \"CREATE ROLE contractor WITH PASSWORD 'contractor' NOSUPERUSER NOCREATEDB NOCREATEROLE LOGIN;\" | psql"
  sudo su postgres -c "createdb -O contractor contractor"

NOTE, for thoes two command you will see::

  could not change directory to "/root": Permission denied

this is from the `su` command, thoes messages can be ignored

Update Packages
~~~~~~~~~~~~~~~

the ubuntu toml package is to old, update with::

  sudo apt install -y python3-pip
  sudo pip3 install toml --upgrade

Configure Apache
~~~~~~~~~~~~~~~~

We will need a HTTP site to serve up static resources, as well as a Proxy server
to bridge from the issloated network.

First create the directory for the static resources::

    mkdir -p /var/www/static

now create the proxy site `/etc/apache2/sites-available/proxy.conf` with the following::

  Listen 3128
  <VirtualHost *:3128>
    ServerName proxy
    ServerAlias proxy.site1.local

    DocumentRoot /var/www/static

    ErrorLog ${APACHE_LOG_DIR}/proxy_error.log
    CustomLog ${APACHE_LOG_DIR}/proxy_access.log combined

    ProxyRequests On
    ProxyVia Full

    CacheEnable disk http://
    CacheEnable disk https://

    NoProxy static static.site1.local
    NoProxy contractor contractor.site1.local
  </VirtualHost>

now create the static site `/etc/apache2/sites-available/static.conf` with the following::

  <VirtualHost *:80>
    ServerName static
    ServerAlias static.site1.local

    DocumentRoot /var/www/static

    LogFormat "%a %t %D \"%r\" %>s %I %O \"%{Referer}i\" \"%{User-Agent}i\" %X" static_log
    ErrorLog ${APACHE_LOG_DIR}/static_error.log
    CustomLog ${APACHE_LOG_DIR}/static_access.log static_log
  </VirtualHost>

Modify `/etc/apache2/sites-available/contractor.conf` and enable the ServerAlias
line, and change the `<domain>` to `site1.local`

Now enable the proxy and static site, disable the default site, and reload the
apache configuration::

  sudo a2ensite proxy
  sudo a2ensite static
  sudo a2dissite 000-default
  sudo a2enmod proxy proxy_connect proxy_ftp proxy_http cache_disk cache
  sudo systemctl restart apache2
  sudo systemctl start apache-htcacheclean

Setup the database
~~~~~~~~~~~~~~~~~~

Now to create the db::

  /usr/lib/contractor/util/manage.py migrate

Install base os config::

  sudo respkg -i contractor-os-base_0.1.respkg
  sudo respkg -i contractor-ubuntu-base_0.1.respkg

Now to enable plugins.
We use manual for misc stuff that is either pre-configured or handled by something else::

  sudo respkg -i contractor-plugins-manual_0.1.respkg

if you are using esx/vcenter::

  sudo respkg -i contractor-plugins-vcenter_0.1.respkg

if you are using virtualbox::

  sudo respkg -i contractor-plugins-virtualbox_0.1.respkg

do manual plugin again so it can cross link to the other plugins::

  sudo respkg -i contractor-plugins-manual_0.1.respkg

Now to setup some base info, and configure bind::

  sudo /usr/lib/contractor/setup/setupWizzard --no-ip-reservation

And now to create a user for us to login as for the API calls::

  /usr/lib/contractor/util/manage.py createsuperuser

that command will ask for a username, email and password.  The email address
does not need to be a real address.

Environment Setup
~~~~~~~~~~~~~~~~~

We will be using the HTTP API to inject new stuff into contractor.
You can run these commands from either the contractor VM, or any place that can make
http requests to contractor.

we will be using curl, make sure it is installed::

  sudo apt install -y curl

First we will define some Environment values so we don't have to keep tying redundant info
the Contractor server, this is assuming you will be running these commands from
the contractor VM, if you are running these steps from someplace else, update the
ip address to the ip address of the contractor vm::

  export COPS=( --header "CInP-Version: 0.9" --header "Content-Type: application/json" )
  export SITE="/api/v1/Site/Site:site1:"
  export CHOST="http://127.0.0.1"

now we need to login::

  cat << EOF | curl "${COPS[@]}" --data @- -X CALL $CHOST/api/v1/Auth/User\(login\)
  { "username": "root", "password": "root" }
  EOF

which will return a auth token, save that to our headers::

  COPS+=( --header "Auth-Id: root")
  COPS+=( --header "Auth-Token: < put auth token from login here>" )

Let's make sure our login is working::

  cat << EOF | curl "${COPS[@]}" --data @- -X CALL $CHOST/api/v1/Auth/User\(whoami\)
  {}
  EOF

that should output::

  "root"

Network Configuration
~~~~~~~~~~~~~~~~~~~~~

The setupWizzard has pre-loaded the database with a stand in host to represent
the contractor VM and has flagged it as pre-built.  It has also created
a site called `site1` and some base DNS configuration. It also took the network
of the primary interface and loaded it into the database named 'main'.

We need to create another address block for the internal network::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Utilities/AddressBlock
  { "site": "$SITE", "name": "internal", "subnet": "10.0.0.1", "gateway_offset": 1, "prefix": "24" }
  EOF

which should output something like::

  {"gateway_offset": 1, "_max_address": "10.0.0.255", "size": "254", "created": "2019-02-23T14:15:06.830987+00:00", "isIpV4": "True", "netmask": "255.255.255.0", "site": "/api/v1/Site/Site:site1:", "gateway": "10.0.0.1", "prefix": 24, "name": "internal", "subnet": "10.0.0.0", "updated": "2019-02-23T14:15:06.830966+00:00"}

Now to add the internal ip of the contractor host::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Utilities/Address
  { "networked": "/api/v1/Utilities/Networked:1:", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "interface_name": "eth1", "offset": 10 }
  EOF

result::

  {"netmask": "255.255.255.0", "ip_address": "10.0.0.10", "created": "2019-02-23T16:20:56.567650+00:00", "pointer": null, "vlan": 0, "networked": "/api/v1/Utilities/Networked:1:", "network": "10.0.0.0", "is_primary": false, "type": "Address", "interface_name": "eth1", "offset": 10, "address_block": "/api/v1/Utilities/AddressBlock:internal:", "gateway": "10.0.0.1", "sub_interface": null, "updated": "2019-02-23T16:20:56.567606+00:00", "prefix": "24"}

now to reserve some ip addresses so they do not get auto assigned::

  for OFFSET in 2 3 4 5 6 7 8 9 11 12 13 14 15 16 17 18 19 20; do
  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Utilities/ReservedAddress
  { "address_block": "/api/v1/Utilities/AddressBlock:internal:", "offset": "$OFFSET", "reason": "Network Reserved" }
  EOF
  done

result::

  {"ip_address": "10.0.0.2", "offset": 2, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.312992+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.312941+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.3", "offset": 3, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.327090+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.327065+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.4", "offset": 4, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.339957+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.339924+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.5", "offset": 5, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.352559+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.352535+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.6", "offset": 6, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.365187+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.365162+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.7", "offset": 7, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.378354+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.378327+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.8", "offset": 8, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.390835+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.390812+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.9", "offset": 9, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.404003+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.403980+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.11", "offset": 11, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.416552+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.416528+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.12", "offset": 12, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.429354+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.429332+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.13", "offset": 13, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.442067+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.442043+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.14", "offset": 14, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.455041+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.455018+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.15", "offset": 15, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.467245+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.467222+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.16", "offset": 16, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.479525+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.479503+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.17", "offset": 17, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.492109+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.492083+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.18", "offset": 18, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.504386+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.504363+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.19", "offset": 19, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.517128+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.517105+00:00", "type": "ReservedAddress"}
  {"ip_address": "10.0.0.20", "offset": 20, "reason": "Network Reserved", "created": "2019-02-23T16:34:54.529458+00:00", "address_block": "/api/v1/Utilities/AddressBlock:internal:", "updated": "2019-02-23T16:34:54.529435+00:00", "type": "ReservedAddress"}

Starting DNS
~~~~~~~~~~~~

Restart bind with new zones::

  sudo systemctl restart bind9

This VM needs to use the contractor generated dns, so edit
`/etc/network/interfaces` to set the dns server to 127.0.0.1
then, reload networking configuration::

  sudo systemctl restart networking

now take a look at the contractor ui at http://<contractor ip>, (this ip is the ip
you assigned to the first interface)

Subcontractor
-------------

install tfptd (used for PXE booting) and the PXE booting agent::

  sudo apt install -y tftpd-hpa
  sudo respkg -i contractor-ipxe_0.1.respkg

now edit `/etc/subcontractor.conf`
enable the modules you want to use, remove the ';' and set the 0 to a 1.
The 1 means one task for that plugin at a time.  If you want things to go faster,
you can try 2 or 4 depending on the plugin, the resources of your vm, etc. In the
dhcpd section, make sure interface and tftp_server are correct, tftp_server
should be the ip of the vm on the new internal interface.

now start up subcontractor::

  sudo systemctl start subcontractor
  sudo systemctl start dhcpd

make sure it's running::

  sudo systemctl status subcontractor
  sudo systemctl status dhcpd

optional, edit `/etc/default/tftpd-hpa` and add '-v ' to TFTP_OPTIONS.  This will
cause tfptd to log transfers to syslog.  This can be helpful in troubleshooting
boot problems. Make sure to run `systemctl restart tftpd-hpa` to reload.

Setting up VM Host
------------------

First we need to make a pre-built entry on a manual foundation to represent the
virtualbox/vcenter/esx host, first creating the foundation::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Manual/ManualFoundation
  { "site": "$SITE", "locator": "host", "blueprint": "/api/v1/BluePrint/FoundationBluePrint:manual-foundation-base:" }
  EOF

which should output something like::

  {"state": "planned", "id_map": null, "located_at": null, "class_list": "['Metal', 'VM', 'Container', 'Switch', 'Manual']", "blueprint": "/api/v1/BluePrint/FoundationBluePrint:manual-foundation-base:", "created": "2019-02-23T16:48:53.818982+00:00", "built_at": null, "locator": "host", "updated": "2019-02-23T16:48:53.818962+00:00", "site": "/api/v1/Site/Site:site1:", "type": "Manual"}

Now to create the structure::

  cat << EOF | curl -i "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Building/Structure
  { "site": "$SITE", "foundation": "/api/v1/Building/Foundation:host:", "hostname": "host", "blueprint": "/api/v1/BluePrint/StructureBluePrint:manual-structure-base:", "auto_build": false }
  EOF

which should output something like::

  HTTP/1.1 201 CREATED
  Date: Sat, 23 Feb 2019 16:49:20 GMT
  Server: Apache/2.4.18 (Ubuntu)
  Object-Id: /api/v1/Building/Structure:2:
  Cinp-Version: 0.9
  Cache-Control: no-cache
  Access-Control-Expose-Headers: Method, Type, Cinp-Version, Count, Position, Total, Multi-Object, Object-Id, Id-Only
  Verb: CREATE
  Access-Control-Allow-Origin: *
  Content-Length: 412
  Content-Type: application/json;charset=utf-8

  {"config_uuid": "349c8a47-e123-4234-91de-c387a440ffa5", "auto_build": false, "hostname": "host", "created": "2019-02-23T16:49:20.064258+00:00", "state": "planned", "blueprint": "/api/v1/BluePrint/StructureBluePrint:manual-structure-base:", "built_at": null, "foundation": "/api/v1/Building/Foundation:host:", "config_values": {}, "updated": "2019-02-23T16:49:20.064239+00:00", "site": "/api/v1/Site/Site:site1:"}

look for the header `Object-Id: /api/v1/Building/Structure:2:`, take note of the
sturcture id (the number between the `:`, in this case 2).

now we need to tell contractor it is allready built so it dosen't try to build it
again.  There curently isn't a API endpoint to manipluate the state of targets,
so we will use a command line utility, this command needs to be run on the
contractor VM. replace `<structure id>` with the id from the previous step::

  /usr/lib/contractor/util/boss -s <structure id> --built

which will output something like this::

  Working with "Structure #2(host) of "manual-structure-base" in "site1""
  No Job to Delete
  Structure #2(host) of "manual-structure-base" in "site1" now set to built.

now to set the ip address, this is the ip address of virtualbox or vcenter/esx host.
This ip will be used by subcontractor to manipluate vms, and will need to be
routeable from the contractor vm, this assumes that address is in the address space
of the contractor vm, specifically the network that setupWizzard created, change
`< offset >` to the offset of the host's ip in that network.  If the ip
address of the host is 192.168.0.52 the setupWizzard assumed you were in a /24
so the offset is `52`, replace structure id with the id from the structure creation
step::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Utilities/Address
  { "networked": "/api/v1/Utilities/Networked:< structure id >:", "address_block": "/api/v1/Utilities/AddressBlock:main:", "interface_name": "eth0", "offset": < offset >, "is_primary": true }
  EOF

which should output something like::

  {"netmask": "255.255.255.0", "updated": "2019-02-23T18:51:53.521628+00:00", "type": "Address", "prefix": "24", "vlan": 0, "ip_address": "192.168.13.22", "interface_name": "eth0", "network": "192.168.13.0", "sub_interface": null, "address_block": "/api/v1/Utilities/AddressBlock:main:", "is_primary": false, "offset": 22, "pointer": null, "gateway": "192.168.13.1", "created": "2019-02-23T18:51:53.521652+00:00", "networked": "/api/v1/Utilities/Networked:2:"}

Now to define the foundation blueprint and create the complex.

VCenter
~~~~~~~

Environment setup::

  export FBP="/api/v1/BluePrint/FoundationBluePrint:vcenter-vm-base:"
  export FMDL="/api/v1/VCenter/VCenterFoundation"
  export FDATA=', "vcenter_host": "/api/v1/VCenter/VCenterComplex:demovcenter:"'

First create the VirtualBox Complex, replace `< datacenter >` with the name of
the VCenter datacenter to put the VMs in, if using ESX directly put 'ha-datacenter',
replace `< cluster >` with the name of the cluster to put the vms in, if using
ESX put the hostname of the ESX server, if it's still default it will be 'localhost.'.
Replace `< structure id >`
with the strudture id from the host creation above, `< username >` and `< password >`
replace with the ESX/VCenter username and password::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/VCenter/VCenterComplex
  { "site": "$SITE", "name": "demovcenter", "description": "Demo VCenter/ESX Host/Complex", "vcenter_datacenter": "< datacenter >", "vcenter_cluster": "< cluster >", "vcenter_host": "/api/v1/Building/Structure:< structure id>:", "vcenter_username": "< username >", "vcenter_password": "< password >" }
  EOF

should return something like::

  {"built_percentage": 90, "state": "planned", "site": "/api/v1/Site/Site:site1:", "created": "2019-02-23T23:51:33.613222+00:00", "vcenter_host": "/api/v1/Building/Structure:2:", "vcenter_password": "vmware123!!", "updated": "2019-02-23T23:51:33.613199+00:00", "vcenter_cluster": null, "name": "demovcenter", "description": "Demo VCenter/ESX Host/Complex", "vcenter_datacenter": "ha-datacenter", "type": "VCenter", "members": [], "vcenter_username": "root"}

Techinically if you are using VCenter, you should create another manual structure
so Contractor knows the hosts of the VCenter cluster, however, for the sake of
simplicity, we will just add the ESX Host/VCenter cluster we just added as the host
of the VCenterCluster as it's only member,  once again the `< structure id >` is
the id of the manual structure  we have been using so far::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Building/ComplexStructure
  { "complex": "/api/v1/Building/Complex:demovcenter:", "structure": "/api/v1/Building/Structure:< structure id>:" }
  EOF

should return something like::

  {"created": "2019-02-24T00:02:06.164123+00:00", "complex": "/api/v1/Building/Complex:demovcenter:", "structure": "/api/v1/Building/Structure:2:", "updated": "2019-02-24T00:02:06.164082+00:00"}

VirtualBox
~~~~~~~~~~

Environment setup::

  export FBP="/api/v1/BluePrint/FoundationBluePrint:virtualbox-vm-base:"
  export FMDL="/api/v1/VirtualBox/VirtualBoxFoundation"
  export FDATA=', "virtualbox_host": "/api/v1/VirtualBox/VirtualBoxComplex:demovbox:"'

First create the VirtualBox Complex::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/VirtualBox/VirtualBoxComplex
  { "site": "$SITE", "name": "demovbox", "description": "Demo VirtualBox Host/Complex" }
  EOF

should output something like::

  {"state": "planned", "description": "Demo VirtualBox Host/Complex", "name": "demovbox", "type": "VirtualBox", "members": [], "built_percentage": 90, "created": "2019-02-20T04:52:30.070436+00:00", "site": "/api/v1/Site/Site:site1:", "updated": "2019-02-20T04:52:30.070407+00:00"}

Now we add the structure host we manually created as a member of the complex,
replace `< structure id >` with the id from the manul host structure from above::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Building/ComplexStructure
  { "complex": "/api/v1/Building/Complex:demovbox:", "structure": "/api/v1/Building/Structure:< structure id>:" }
  EOF

should output something like::

  {"complex": "/api/v1/Building/Complex:demovbox:", "structure": "/api/v1/Building/Structure:2:", "created": "2019-02-20T04:55:31.730431+00:00", "updated": "2019-02-20T04:55:31.730357+00:00"}

Contractor is now running, now let's configure it to make a VM.

Creating a VM
-------------

Now we create the Foundation of the VM to be created::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/$FMDL
  { "site": "$SITE", "locator": "testvm01", "blueprint": "$FBP" $FDATA }
  EOF

output::

  {"state": "planned", "site": "/api/v1/Site/Site:site1:", "type": "VirtualBox", "id_map": "", "virtualbox_host": "/api/v1/VirtualBox/VirtualBoxComplex:demovbox:", "blueprint": "/api/v1/BluePrint/FoundationBluePrint:virtualbox-vm-base:", "built_at": null, "locator": "tesvm01", "located_at": null, "updated": "2019-02-20T04:58:52.855473+00:00", "created": "2019-02-20T04:58:52.855507+00:00", "class_list": "['VM', 'VirtualBox']", "virtualbox_uuid": null}

create interface::

  cat << EOF | curl "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Utilities/RealNetworkInterface
  { "foundation": "/api/v1/Building/Foundation:testvm01:", "name": "eth0", "physical_location": "eth0", "is_provisioning": true }
  EOF

output::

  {"pxe": null, "name": "eth0", "is_provisioning": true, "physical_location": "eth0", "updated": "2019-02-25T14:28:36.245466+00:00", "mac": null, "foundation": "/api/v1/Building/Foundation:testvm01:", "created": "2019-02-25T14:28:36.245500+00:00"}

Now we will create a VM with the Ubuntu Bionic blueprint::

  cat << EOF | curl -i "${COPS[@]}" --data @- -X CREATE $CHOST/api/v1/Building/Structure
  { "site": "$SITE", "foundation": "/api/v1/Building/Foundation:testvm01:", "hostname": "testvm01", "blueprint": "/api/v1/BluePrint/StructureBluePrint:ubuntu-bionic-base:", "auto_build": true }
  EOF

once again take node of the structure id.  Now we assign and ip address, we will
let contractor pick, we are going to use the helper method `nextAddress`.  Replace
`< structure id >` with the structure id from the previous call::

  cat << EOF | curl "${COPS[@]}" --data @- -X CALL "$CHOST/api/v1/Utilities/AddressBlock:internal:(nextAddress)"
  { "structure": "/api/v1/Building/Structure:< structure id >:", "interface_name": "eth0", "is_primary": true }
  EOF

output::

  "/api/v1/Utilities/Address:30:"


Ok, we have a VM in the database, now to see it get built.  Pull up the `http://<contractor ip>`
in a web browser if you don't have it open allready, go to the `Job Log` should see an
entry saying that the foundation build has started.  Goto the `Jobs` should see a Foundation
or Structure Job there.  The Foundation Job won't last long.  In the top right of the
page is a refresh and auto refresh buttons.
