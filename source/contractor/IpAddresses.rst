Ip Addresses
============

Ip Addresses come in a few flavors, there is a BaseAddress class which all Ip Addresses
belong to that defines an Ip Address as an Offset in an AddressBlock.  The flavors
of BaseAddress are **Address** - an Address that can belong to **Networked**.
**ReservedAddress** - an Address that is reserved by something outside Contractor's
scope.  And **DynamicAddress** - an Address that belongs to a DHCP group.

An **AddressBlock** is defined by its network and prefix and can optionally
hold a gateway offset.  No two AddressBlocks can overlap.  An AddressBlock is
also tied to a Site.

**Address** can belong to anything that is **Networked**. Structures are Networked,
as well as some Foundations like the IPMIFoundation.  An Address enforces that the
Site the Networked belongs to is the same as the AddressBlock the Address Belongs to,
thereby simplifying management, helping to assure that the address will work.  Sites
can also hold network configuration information (such as DNS servers and DNS
search zones) which can be Site specfic.  Address also adds optional vLAN value, as
well as an optional sub-interface value.  In some cases (such as with containers),
a Structure doesn't have it's own IpAddress, but relies on its host IpAddress.  In
these cases, Address can be configured to point to another Address.  Address also
has a binary value is_primary, which defines which of all the potential Addresses
this Networked device is to be used as it's primary DNS name, as well as the Address
to use when referring to it.  NOTE: the site is not checked through the pointer
field, this way the host and the hosted can belond to diferent sites, make sure
your site and network configuration works for this.

**ReservedAddress** adds a single field which is a description of the reason
the Address has been reserved.

**DynamicAddress** adds a PXE value which is used to PXE boot what ever device
gets this lease and wants to PXE boot.  This is optional.

Related to Addresses is the **NetworkInterface**.  A NetworkInterface is a named
connection between a Networked and a set of IpAddressed.  A NetworkInterface has a
flag is_provisioning which indicates which interface should be used for communication
during provisioning.  During provisioning only the primary IP on the provisioning
interface is used.  Not until the final Structure(i.e., Operating System) is installed
and configured will the other interfaces and Ip Addresses be used.  NetworkInterface
has three flavors, **RealNetworkInterface**, **AbstractNetworkInterface**, and
**AggregatedNetworkInterface**.

**RealNetworkInterface** is to identify physical ports, (or in case of things like
Blades/VMs, Real as far as the OS/BIOS is concerned).  This type requires a MAC address
and is PXE bootable.

**AbstractNetworkInterface** is for interfaces that do not "physically" exist, like
internal bridge interfaces, or loop back interfaces.

**AggregatedNetworkInterface** is for grouping multiple NetworkInterfaces together
into a single AbstractNetworkInterface.  This is for things such as Port Channels,
Bonded Interfaces, LACP, etc.  One interface is designated as the master interface.
When the final Structure is not installed and configured, this is the interface
that is used.  There is also a list of the other interfaces that are grouped
in as well as a Key/Value field to store configuration information (such as
the bonding mode).

NOTE: All networking information is combined together and added to the Configuration
Information of the Structure/Foundation as a whole.

NOTE2: Technically speaking other than dedicated NetworkInterfaces, such as the IPMI
port on the IPMIFoundation, Foundations do not have Ip Addresses.  Thus most Physical
Foundations will borrow the Address information of the Structure that they are configured
with to do tasks of the Foundation Jobs.  Without a Structure, Foundation Jobs that
require an Address can not be done. (This is something that will hopefully change
in the future by borrowing from a dynamic pool, but for now a Structure is required.)
