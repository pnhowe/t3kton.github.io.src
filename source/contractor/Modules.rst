Modules
=======

BluePrint
---------

Contains BluePrint, a generic base class for StructureBluePrint and FoundationBluePrint.
Also contains the Scripts and linkages between the BluePrints and Scripts.  It also contains
PXE, things to which PXE bootable devices can be set to boot to.

Building
--------

Contains Foundation, a generic base class for all Foundations provided by plugins.
Structure, the class for Structures that go on Foundations.  Complex, a
grouping of Foundations (ie: a cluster).  Dependency, allows Foundations to
depend on Structures and/or jobs to be complete on a structure.

Foreman
-------

Contains BaseJob a generic base class for FoundationJob, StructureJob and DependancyJob,
which are jobs that are inflight for Foundations, Structures, and Dependancies
respectivly.

Site
----

Contains Site, for grouping Foundations and Structures into logical groups for
easier management.

SubContractor
-------------

Interface for Subcontractor, probably going to get moved to Foreman

User
----

Handles Contractor Users.

Utilities
---------

Handles Network Interfaces, Ip Addresses, and Power interfaces (Not all flushed out yet)


lib
---

Common utilties for all of Contractor

tscript
-------

T3kton Script parser and runner
