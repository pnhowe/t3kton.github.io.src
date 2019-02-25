Introduction to Contractor
==========================

The Goal of Contractor is to provide a Universal Interface/API to Automate
Management, Provisioning, Deploying, and Configuration of Resource.

Contractor uses building terms (for the most part) to try to avoid name
colisions with various platforms and systems.

First two terms are **Foundation** and **Structure**.  A Foundation is something
to build on, a Structure is the thing you want to build.  Say we want to
build a Web Server.  Our Structure is a set of configuration values and scripts
required to build that web server, i.e., Install Ubuntu LTS 18.04 and install and
Configure Apache (more on thoes details in .....).  The Foundation is what we
want to install that on, i.e., a VM, Container, Blade Server, Baremetal Server,
Rasbery PI, ECS, GCD, etc...  Not all Foundations are going to be compatible
with all Structures, however a well defined Structure can be installed on most
of them.

So far we have this::

  +-----------------------------+
  |                             |
  |  Structure (Web Server)     |
  |                             |
  +-----------------------------+
               |
               |
  +-----------------------------+
  |                             |
  |   Foundation                |
  |                             |
  +-----------------------------+


The next term is **Site**.  A site is a Logical grouping of things.  Let's put
our Web Server in a Site called "Cluster 1"

::

  +-------------------------------------+
  |  Cluster 1                          |
  |                                     |
  |    +-----------------------------+  |
  |    |                             |  |
  |    |  Structure (Web Server)     |  |
  |    |                             |  |
  |    +------------+----------------+  |
  |                 |                   |
  |                 |                   |
  |    +------------+----------------+  |
  |    |                             |  |
  |    |   Foundation                |  |
  |    |                             |  |
  |    +-----------------------------+  |
  |                                     |
  +-------------------------------------+

Each Item we have used so far contains configuration values.  These are Key
Value pairs that can be overlayed.  In this case Contractor will take the
configuration values of "Cluster 1" then overlay them with "Foundation" and
the "Structure".

Sites can be be put into other Sites.  For example we have Clusters 1, 2, and 3
in "Datacenter West".

::

  +---------------------------------------------------------------------------------------------+
  | Datacenter West                                                                             |
  |                                                                                             |
  |  +-------------------------------------+ +----------------------+ +----------------------+  |
  |  |  Cluster 1                          | |  Cluster 2           | |  Cluster 3           |  |
  |  |                                     | |                      | |                      |  |
  |  |    +-----------------------------+  | |    +--------------+  | |    +--------------+  |  |
  |  |    |                             |  | |    |              |  | |    |              |  |  |
  |  |    |  Structure (Web Server)     |  | |    |  Structure   |  | |    |  Structure   |  |  |
  |  |    |                             |  | |    |              |  | |    |              |  |  |
  |  |    +------------+----------------+  | |    +-----+--------+  | |    +-----+--------+  |  |
  |  |                 |                   | |          |           | |          |           |  |
  |  |                 |                   | |          |           | |          |           |  |
  |  |    +------------+----------------+  | |    +-----+--------+  | |    +-----+--------+  |  |
  |  |    |                             |  | |    |              |  | |    |              |  |  |
  |  |    |   Foundation                |  | |    |  Foundation  |  | |    |  Foundation  |  |  |
  |  |    |                             |  | |    |              |  | |    |              |  |  |
  |  |    +-----------------------------+  | |    +--------------+  | |    +--------------+  |  |
  |  |                                     | |                      | |                      |  |
  |  +-------------------------------------+ +----------------------+ +----------------------+  |
  |                                                                                             |
  +---------------------------------------------------------------------------------------------+

Now the configuration information will first have site "Datacenter West" then,
Cluster X, Foundation, Structure.  This comes in handy for propagating configuration
information without having to set it for each item individually.  For example,
we can have the DNS Search Zones be set to "west.site.com" in the site "Datacenter West"
and prepend that with "cluster1.site.com" in "Cluster 1".  If at any time we want
some other global DNS search zone, we add it to the top and it automatically propagates
down.  You could also set "Release"="Prod" in "Datacenter West" and then create a
"Cluster Test" and override the "Release" to the value "Test".  You could also do
A-B testing, etc.

Any Item can make an http request to Contractor and Contractor will reply with a JSON
encoded reply with that item's combined configuration values.

This is all fun and all, but not really useful.  Let's change things up a bit and
install ESX on the baremetal and put a few Web servers on ESX.

Before we do that we need to dig into Foundations a little more. The **Foundation**
class is meant as a root class for specific target handlers to work against.

We are going to use the **IPMIFoundation** to handle the baremetal machines on which
we are installing ESX on, and **VCenterFoundation** to handle the vms on the
ESX/VCenter.

Note: we are going to omit Cluster 2 and 3 for now, they are clones of Cluster 1::

  +-----------------------------------------------------------------------------+
  | Datacenter West                                                             |
  |                                                                             |
  |  +-----------------------------------------------------------------------+  |
  |  |  Cluster 1                                                            |  |
  |  |                                                                       |  |
  |  |  +-----------------------------+ +-----------------------------+      |  |
  |  |  |                             | |                             |      |  |
  |  |  |  Structure (Web Server)     | |  Structure (Web Server)     |      |  |
  |  |  |                             | |                             |      |  |
  |  |  +------------+----------------+ +------------+----------------+      |  |
  |  |               |                               |                       |  |
  |  |               |                               |                       |  |
  |  |  +------------+----------------+ +------------+----------------+      |  |
  |  |  |                             | |                             |      |  |
  |  |  |   VCenterFoundation         | |   VCenterFoundation         |      |  |
  |  |  |                             | |                             |      |  |
  |  |  +------------------------+----+ +---+-------------------------+      |  |
  |  |                           |          |                                |  |
  |  |                      +----+----------+---+                            |  |
  |  |                      |                   |                            |  |
  |  |                      | VCenter Complex   |                            |  |
  |  |                      |                   |                            |  |
  |  |                      +--------+----------+                            |  |
  |  |                               |                                       |  |
  |  |                  +------------+----------------+                      |  |
  |  |                  |                             |                      |  |
  |  |                  |  Structure (ESX)            |                      |  |
  |  |                  |                             |                      |  |
  |  |                  +------------+----------------+                      |  |
  |  |                               |                                       |  |
  |  |                               |                                       |  |
  |  |                  +------------+----------------+                      |  |
  |  |                  |                             |                      |  |
  |  |                  |   IPMIFoundation            |                      |  |
  |  |                  |                             |                      |  |
  |  |                  +-----------------------------+                      |  |
  |  |                                                                       |  |
  |  +-----------------------------------------------------------------------+  |
  |                                                                             |
  +-----------------------------------------------------------------------------+

This introduces our next item the **Complex** as in a building complex.  A Complex
is a group of Structures providing something for more Foundations to be built on.
A Complex (depending on the type) can have one or more Structures as members.
NOTE: the configuration info of the Structure and Foundations that make up a
cluster do **NOT** flow through to the Foundations and Structures built on that
complex.  The Members of the Complex can even belong to another site.

For Example::

  +-----------------------------------------------------------------------------+
  | Datacenter West                                                             |
  |                                                                             |
  |  +-----------------------------------------------------------------------+  |
  |  |  Cluster 1                                                            |  |
  |  |                                                                       |  |
  |  |  +-----------------------------+ +-----------------------------+      |  |
  |  |  |                             | |                             |      |  |
  |  |  |  Structure (Web Server)     | |  Structure (Web Server)     |      |  |
  |  |  |                             | |                             |      |  |
  |  |  +------------+----------------+ +------------+----------------+      |  |
  |  |               |                               |                       |  |
  |  |               |                               |                       |  |
  |  |  +------------+----------------+ +------------+----------------+      |  |
  |  |  |                             | |                             |      |  |
  |  |  |   VCenterFoundation         | |   VCenterFoundation         |      |  |
  |  |  |                             | |                             |      |  |
  |  |  +------------------------+----+ +---+-------------------------+      |  |
  |  |                           |          |                                |  |
  |  +-----------------------------------------------------------------------+  |
  |  |                           |          |                                |  |
  |  |  Cluster 1 Hosting   +----+----------+---+                            |  |
  |  |                      |                   |                            |  |
  |  |                      | VCenter Complex   |                            |  |
  |  |                      |                   |                            |  |
  |  |                      +---+-------------+-+                            |  |
  |  |                          |             |                              |  |
  |  |                          |             |                              |  |
  |  |                          |             |                              |  |
  |  |                          |             |                              |  |
  |  |     +--------------------+------+   +--+-------------------------+    |  |
  |  |     |                           |   |                            |    |  |
  |  |     | Structure (ESX)           |   | Structure (ESX)            |    |  |
  |  |     |                           |   |                            |    |  |
  |  |     +----------+----------------+   +-----------+----------------+    |  |
  |  |                |                                |                     |  |
  |  |                |                                |                     |  |
  |  |     +----------+----------------+   +-----------+----------------+    |  |
  |  |     |                           |   |                            |    |  |
  |  |     |  IPMIFoundation           |   |  IPMIFoundation            |    |  |
  |  |     |                           |   |                            |    |  |
  |  |     +---------------------------+   +----------------------------+    |  |
  |  |                                                                       |  |
  |  +-----------------------------------------------------------------------+  |
  |                                                                             |
  +-----------------------------------------------------------------------------+

Complexes also cause Contractor to build the Web Server Structure/Foundations
after the ESX Structure/Foundations are done.  Also the example would look pretty
much the same for a Docker/OpenStack/etc Complex.

Side Track to the Manefesto
---------------------------

At this point you are probably wondering how having all these Foundation types
is simplifying deployments.  By separating the configuration of the "Hosted" and
the "Host" we can effectively divide up the job of configuraing the sytem.  (Do
I get to drop the DevOps Buzzword now?)  As a Developer/Engineer configures their
code, they embody that in a Structure.  They can package that configuration
information along with their code/designs and that configuration can also
be tested and verified via CICD and similar work flows.  This way the very
same configuration information is for all stages of deployment.  It is true
that some Foundations require different considerations, however a well designed
Structure Configuration can work for Containers (and the like) as well as
OS installers (Baremetal/VM/Blade/AWS, etc.)  Now when the Operations people need to
turn it up to 11 (or 12) they just pick the location to deploy and no matter if
it is hosted on prem in VMs, or deployed to AWS for some peak load handling,
Operations can scale as needed, to whatever.

Also by allowing every thing, no matter the platform, to be tracked in the same
place, you now have a single source of truth for your monitoring sytem to rely on.
You don't have to worry about parts of your Micro Services failing to auto-register.
And, you know exactly what is deployed where; useful when hardware needs to be
swapped out.

Your Operations teams are also free to try changing out hosting solutions without
retooling everything to try it -- in some cases without involving Engineering
to do so.

Not only can you unify your provisioning tools, but also the auto-scaling tools.

You are also free from vendor lock in.  If a new Cloud provider comes along, they
don't need to have an AWS like API to use them, just a Foundation subclass
provider that talks to that Cloud provider's API and you are set.  Same if
a new class of hardware comes along (ARM servers anyone ;-) ) or a new way of
approaching hosting (the next thing after containers).  You can truly be
"serverless" (I know another buzzword, meaning your deployments are agnostic
as to the technology they are being deployed on).  And you don't have to try to
fit all your use cases into one silver bullet.  You can have a nice auto-scaling
Container Cloud/Swarm with your micro services right next to standard VMs running
the databases and object storage.  All with one "pane of glass"

Ok, back to business, buzzword dropping disabled...

Back to business
----------------

One final piece of the deployment puzzle, the **Dependency**.  This is to make sure
your deployments happen in order.  For example, you can't install any OSes until
the Switch is provisioned.  Also you may have to allocate space on an NFS mount
before installing a VM.  This is where Dependencies come in, allowing a Foundation
to Depend on a Structure being built, and/or a job being run on a Structure.


BluePrints
----------

Now that we have talked about the parts, we need to talk about how those things
are confugred and that is handled by **BluePrint**, specifically the
**FoundationBluePrint** and the **StructureBluePrint**.  A Blueprint also holds
cocnfiguration values, as well as links to scripts which are executed when the
Structure/Foundation that blueprint is for is configured, destroyed, or had a named
script run on it.  The BluePrint is the thing that Engineering and Operations
build to embody the process and configuration information of Creating the
Structure/Foundation.

Other
-----

There are other Classes/Components in Contractor, but they are mostly for dealing
with Configure/Destroy/Misc Jobs (the Forman module).  As well as keeping track
of Ip Addresses and other "Utilities".  Those are documented elsewhere.
