Vcenter/ESX Setup
=================

For the puropses of this demo, VCenter it's self is not required.  All you will
need is a standalone ESX server.  To prevent Contractor from DHCPing your other
systems, you will want to create a private network to build contractor's targets in.
This demo assumes it will be `10.0.0.1/24`.

After you create the network/port group, add a second interface to the contractor VM on this
new network, and assign the ip `10.0.0.10` to that interface.

In the `/etc/subcontractor.conf` file under the `dhcpd` section, set
the `listen_interface` to the name of the newly created interface.

you will need to install the virtualbox python bindings either use::

  sudo pip3 install pyvmomi

or::

  apt install python3-pyvmomi
