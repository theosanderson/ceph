==========================
 Monitor Config Reference
==========================

Understanding how to configure a :term:`Ceph Monitor` is an important part of
building a reliable :term:`Ceph Storage Cluster`. **All Ceph Storage Clusters
have at least one monitor**. The monitor complement usually remains fairly
consistent, but you can add, remove or replace a monitor in a cluster. See
`Adding/Removing a Monitor`_ for details.


.. index:: Ceph Monitor; Paxos

Background
==========

Ceph Monitors maintain a "master copy" of the :term:`Cluster Map`, which means a
:term:`Ceph Client` can determine the location of all Ceph Monitors, Ceph OSD
Daemons, and Ceph Metadata Servers just by connecting to one Ceph Monitor and
retrieving a current cluster map. Before Ceph Clients can read from or write to
Ceph OSD Daemons or Ceph Metadata Servers, they must connect to a Ceph Monitor
first. With a current copy of the cluster map and the CRUSH algorithm, a Ceph
Client can compute the location for any object. The ability to compute object
locations allows a Ceph Client to talk directly to Ceph OSD Daemons, which is a
very important aspect of Ceph's high scalability and performance. See 
`Scalability and High Availability`_ for additional details.

The primary role of the Ceph Monitor is to maintain a master copy of the cluster
map. Ceph Monitors also provide authentication and logging services. Ceph
Monitors write all changes in the monitor services to a single Paxos instance,
and Paxos writes the changes to a key/value store for strong consistency. Ceph
Monitors can query the most recent version of the cluster map during sync
operations. Ceph Monitors leverage the key/value store's snapshots and iterators
(using leveldb) to perform store-wide synchronization.

.. ditaa::
 /-------------\               /-------------\
 |   Monitor   | Write Changes |    Paxos    |
 |   cCCC      +-------------->+   cCCC      |
 |             |               |             |
 +-------------+               \------+------/
 |    Auth     |                      |
 +-------------+                      | Write Changes
 |    Log      |                      |
 +-------------+                      v
 | Monitor Map |               /------+------\
 +-------------+               | Key / Value |
 |   OSD Map   |               |    Store    |
 +-------------+               |  cCCC       |
 |   PG Map    |               \------+------/
 +-------------+                      ^
 |   MDS Map   |                      | Read Changes
 +-------------+                      |
 |    cCCC     |*---------------------+
 \-------------/


.. deprecated:: version 0.58

In Ceph versions 0.58 and earlier, Ceph Monitors use a Paxos instance for
each service and store the map as a file. 

.. index:: Ceph Monitor; cluster map

Cluster Maps
------------

The cluster map is a composite of maps, including the monitor map, the OSD map,
the placement group map and the metadata server map. The cluster map tracks a
number of important things: which processes are ``in`` the Ceph Storage Cluster;
which processes that are ``in`` the Ceph Storage Cluster are ``up`` and running
or ``down``; whether, the placement groups are ``active`` or ``inactive``, and
``clean`` or in some other state; and, other details that reflect the current
state of the cluster such as the total amount of storage space, and the amount
of storage used.

When there is a significant change in the state of the cluster--e.g., a Ceph OSD
Daemon goes down, a placement group falls into a degraded state, etc.--the
cluster map gets updated to reflect the current state of the cluster.
Additionally, the Ceph Monitor also maintains a history of the prior states of
the cluster. The monitor map, OSD map, placement group map and metadata server
map each maintain a history of their map versions. We call each version an
"epoch."

When operating your Ceph Storage Cluster, keeping track of these states is an
important part of your system administration duties. See `Monitoring a Cluster`_
and `Monitoring OSDs and PGs`_ for additional details.

.. index:: high availability; quorum

Monitor Quorum
--------------

Our Configuring ceph section provides a trivial `Ceph configuration file`_ that
provides for one monitor in the test cluster. A cluster will run fine with a
single monitor; however, **a single monitor is a single-point-of-failure**. To
ensure high availability in a production Ceph Storage Cluster, you should run
Ceph with multiple monitors so that the failure of a single monitor **WILL NOT**
bring down your entire cluster.

When a Ceph Storage Cluster runs multiple Ceph Monitors for high availability,
Ceph Monitors use `Paxos`_ to establish consensus about the master cluster map.
A consensus requires a majority of monitors running to establish a quorum for
consensus about the cluster map (e.g., 1; 2 out of 3; 3 out of 5; 4 out of 6;
etc.).

.. confval:: mon_force_quorum_join

.. index:: Ceph Monitor; consistency

Consistency
-----------

When you add monitor settings to your Ceph configuration file, you need to be
aware of some of the architectural aspects of Ceph Monitors. **Ceph imposes
strict consistency requirements** for a Ceph monitor when discovering another
Ceph Monitor within the cluster. Whereas, Ceph Clients and other Ceph daemons
use the Ceph configuration file to discover monitors, monitors discover each
other using the monitor map (monmap), not the Ceph configuration file.

A Ceph Monitor always refers to the local copy of the monmap when discovering
other Ceph Monitors in the Ceph Storage Cluster. Using the monmap instead of the
Ceph configuration file avoids errors that could break the cluster (e.g., typos
in ``ceph.conf`` when specifying a monitor address or port). Since monitors use
monmaps for discovery and they share monmaps with clients and other Ceph
daemons, **the monmap provides monitors with a strict guarantee that their
consensus is valid.**

Strict consistency also applies to updates to the monmap. As with any other
updates on the Ceph Monitor, changes to the monmap always run through a
distributed consensus algorithm called `Paxos`_. The Ceph Monitors must agree on
each update to the monmap, such as adding or removing a Ceph Monitor, to ensure
that each monitor in the quorum has the same version of the monmap. Updates to
the monmap are incremental so that Ceph Monitors have the latest agreed upon
version, and a set of previous versions. Maintaining a history enables a Ceph
Monitor that has an older version of the monmap to catch up with the current
state of the Ceph Storage Cluster.

If Ceph Monitors were to discover each other through the Ceph configuration file
instead of through the monmap, additional risks would be introduced because
Ceph configuration files are not updated and distributed automatically. Ceph
Monitors might inadvertently use an older Ceph configuration file, fail to
recognize a Ceph Monitor, fall out of a quorum, or develop a situation where
`Paxos`_ is not able to determine the current state of the system accurately.


.. index:: Ceph Monitor; bootstrapping monitors

Bootstrapping Monitors
----------------------

In most configuration and deployment cases, tools that deploy Ceph help
bootstrap the Ceph Monitors by generating a monitor map for you (e.g.,
``cephadm``, etc). A Ceph Monitor requires a few explicit
settings:

- **Filesystem ID**: The ``fsid`` is the unique identifier for your
  object store. Since you can run multiple clusters on the same
  hardware, you must specify the unique ID of the object store when
  bootstrapping a monitor.  Deployment tools usually do this for you
  (e.g., ``cephadm`` can call a tool like ``uuidgen``), but you
  may specify the ``fsid`` manually too.
  
- **Monitor ID**: A monitor ID is a unique ID assigned to each monitor within 
  the cluster. It is an alphanumeric value, and by convention the identifier 
  usually follows an alphabetical increment (e.g., ``a``, ``b``, etc.). This 
  can be set in a Ceph configuration file (e.g., ``[mon.a]``, ``[mon.b]``, etc.), 
  by a deployment tool, or using the ``ceph`` commandline.

- **Keys**: The monitor must have secret keys. A deployment tool such as 
  ``cephadm`` usually does this for you, but you may
  perform this step manually too. See `Monitor Keyrings`_ for details.

For additional details on bootstrapping, see `Bootstrapping a Monitor`_.

.. index:: Ceph Monitor; configuring monitors

Configuring Monitors
====================

To apply configuration settings to the entire cluster, enter the configuration
settings under ``[global]``. To apply configuration settings to all monitors in
your cluster, enter the configuration settings under ``[mon]``. To apply
configuration settings to specific monitors, specify the monitor instance 
(e.g., ``[mon.a]``). By convention, monitor instance names use alpha notation.

.. code-block:: ini

	[global]

	[mon]		
		
	[mon.a]
		
	[mon.b]
		
	[mon.c]


Minimum Configuration
---------------------

The bare minimum monitor settings for a Ceph monitor via the Ceph configuration
file include a hostname and a network address for each monitor. You can configure
these under ``[mon]`` or under the entry for a specific monitor.

.. code-block:: ini

	[global]
		mon host = 10.0.0.2,10.0.0.3,10.0.0.4

.. code-block:: ini

	[mon.a]
		host = hostname1
		mon addr = 10.0.0.10:6789

See the `Network Configuration Reference`_ for details.

.. note:: This minimum configuration for monitors assumes that a deployment 
   tool generates the ``fsid`` and the ``mon.`` key for you.

Once you deploy a Ceph cluster, you **SHOULD NOT** change the IP addresses of
monitors. However, if you decide to change the monitor's IP address, you
must follow a specific procedure. See `Changing a Monitor's IP Address`_ for
details.

Monitors can also be found by clients by using DNS SRV records. See `Monitor lookup through DNS`_ for details.

Cluster ID
----------

Each Ceph Storage Cluster has a unique identifier (``fsid``). If specified, it
usually appears under the ``[global]`` section of the configuration file.
Deployment tools usually generate the ``fsid`` and store it in the monitor map,
so the value may not appear in a configuration file. The ``fsid`` makes it
possible to run daemons for multiple clusters on the same hardware.

.. confval:: fsid

.. index:: Ceph Monitor; initial members

Initial Members
---------------

We recommend running a production Ceph Storage Cluster with at least three Ceph
Monitors to ensure high availability. When you run multiple monitors, you may
specify the initial monitors that must be members of the cluster in order to
establish a quorum. This may reduce the time it takes for your cluster to come
online.

.. code-block:: ini

	[mon]		
		mon_initial_members = a,b,c


.. confval:: mon_initial_members

.. index:: Ceph Monitor; data path

Data
----

Ceph provides a default path where Ceph Monitors store data. For optimal
performance in a production Ceph Storage Cluster, we recommend running Ceph
Monitors on separate hosts and drives from Ceph OSD Daemons. As leveldb uses
``mmap()`` for writing the data, Ceph Monitors flush their data from memory to disk
very often, which can interfere with Ceph OSD Daemon workloads if the data
store is co-located with the OSD Daemons.

In Ceph versions 0.58 and earlier, Ceph Monitors store their data in plain files. This 
approach allows users to inspect monitor data with common tools like ``ls``
and ``cat``. However, this approach didn't provide strong consistency.

In Ceph versions 0.59 and later, Ceph Monitors store their data as key/value
pairs. Ceph Monitors require `ACID`_ transactions. Using a data store prevents
recovering Ceph Monitors from running corrupted versions through Paxos, and it
enables multiple modification operations in one single atomic batch, among other
advantages.

Generally, we do not recommend changing the default data location. If you modify
the default location, we recommend that you make it uniform across Ceph Monitors
by setting it in the ``[mon]`` section of the configuration file.


.. confval:: mon_data
.. confval:: mon_data_size_warn
.. confval:: mon_data_avail_warn
.. confval:: mon_data_avail_crit
.. confval:: mon_warn_on_cache_pools_without_hit_sets
.. confval:: mon_warn_on_crush_straw_calc_version_zero
.. confval:: mon_warn_on_legacy_crush_tunables
.. confval:: mon_crush_min_required_version
.. confval:: mon_warn_on_osd_down_out_interval_zero
.. confval:: mon_warn_on_slow_ping_ratio
.. confval:: mon_warn_on_slow_ping_time
.. confval:: mon_warn_on_pool_no_redundancy
.. confval:: mon_cache_target_full_warn_ratio
.. confval:: mon_health_to_clog
.. confval:: mon_health_to_clog_tick_interval
.. confval:: mon_health_to_clog_interval

.. index:: Ceph Storage Cluster; capacity planning, Ceph Monitor; capacity planning

.. _storage-capacity:

Storage Capacity
----------------

When a Ceph Storage Cluster gets close to its maximum capacity
(see``mon_osd_full ratio``), Ceph prevents you from writing to or reading from OSDs
as a safety measure to prevent data loss. Therefore, letting a
production Ceph Storage Cluster approach its full ratio is not a good practice,
because it sacrifices high availability. The default full ratio is ``.95``, or
95% of capacity. This a very aggressive setting for a test cluster with a small
number of OSDs.

.. tip:: When monitoring your cluster, be alert to warnings related to the 
   ``nearfull`` ratio. This means that a failure of some OSDs could result
   in a temporary service disruption if one or more OSDs fails. Consider adding
   more OSDs to increase storage capacity.

A common scenario for test clusters involves a system administrator removing an
OSD from the Ceph Storage Cluster, watching the cluster rebalance, then removing
another OSD, and another, until at least one OSD eventually reaches the full
ratio and the cluster locks up. We recommend a bit of capacity
planning even with a test cluster. Planning enables you to gauge how much spare
capacity you will need in order to maintain high availability. Ideally, you want
to plan for a series of Ceph OSD Daemon failures where the cluster can recover
to an ``active+clean`` state without replacing those OSDs
immediately. Cluster operation continues in the ``active+degraded`` state, but this
is not ideal for normal operation and should be addressed promptly.

The following diagram depicts a simplistic Ceph Storage Cluster containing 33
Ceph Nodes with one OSD per host, each OSD reading from
and writing to a 3TB drive. So this exemplary Ceph Storage Cluster has a maximum
actual capacity of 99TB. With a ``mon osd full ratio`` of ``0.95``, if the Ceph
Storage Cluster falls to 5TB of remaining capacity, the cluster will not allow
Ceph Clients to read and write data. So the Ceph Storage Cluster's operating
capacity is 95TB, not 99TB.

.. ditaa::
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
 | Rack 1 |  | Rack 2 |  | Rack 3 |  | Rack 4 |  | Rack 5 |  | Rack 6 |
 | cCCC   |  | cF00   |  | cCCC   |  | cCCC   |  | cCCC   |  | cCCC   |
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
 | OSD 1  |  | OSD 7  |  | OSD 13 |  | OSD 19 |  | OSD 25 |  | OSD 31 |
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
 | OSD 2  |  | OSD 8  |  | OSD 14 |  | OSD 20 |  | OSD 26 |  | OSD 32 |
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
 | OSD 3  |  | OSD 9  |  | OSD 15 |  | OSD 21 |  | OSD 27 |  | OSD 33 |
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
 | OSD 4  |  | OSD 10 |  | OSD 16 |  | OSD 22 |  | OSD 28 |  | Spare  | 
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
 | OSD 5  |  | OSD 11 |  | OSD 17 |  | OSD 23 |  | OSD 29 |  | Spare  |
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
 | OSD 6  |  | OSD 12 |  | OSD 18 |  | OSD 24 |  | OSD 30 |  | Spare  |
 +--------+  +--------+  +--------+  +--------+  +--------+  +--------+

It is normal in such a cluster for one or two OSDs to fail. A less frequent but
reasonable scenario involves a rack's router or power supply failing, which
brings down multiple OSDs simultaneously (e.g., OSDs 7-12). In such a scenario,
you should still strive for a cluster that can remain operational and achieve an
``active + clean`` state--even if that means adding a few hosts with additional
OSDs in short order. If your capacity utilization is too high, you may not lose
data, but you could still sacrifice data availability while resolving an outage
within a failure domain if capacity utilization of the cluster exceeds the full
ratio. For this reason, we recommend at least some rough capacity planning.

Identify two numbers for your cluster:

#. The number of OSDs. 
#. The total capacity of the cluster 

If you divide the total capacity of your cluster by the number of OSDs in your
cluster, you will find the mean average capacity of an OSD within your cluster.
Consider multiplying that number by the number of OSDs you expect will fail
simultaneously during normal operations (a relatively small number). Finally
multiply the capacity of the cluster by the full ratio to arrive at a maximum
operating capacity; then, subtract the number of amount of data from the OSDs
you expect to fail to arrive at a reasonable full ratio. Repeat the foregoing
process with a higher number of OSD failures (e.g., a rack of OSDs) to arrive at
a reasonable number for a near full ratio.

The following settings only apply on cluster creation and are then stored in
the OSDMap. To clarify, in normal operation the values that are used by OSDs
are those found in the OSDMap, not those in the configuration file or central
config store.

.. code-block:: ini

	[global]
		mon_osd_full_ratio = .80
		mon_osd_backfillfull_ratio = .75
		mon_osd_nearfull_ratio = .70


``mon_osd_full_ratio`` 

:Description: The threshold percentage of device space utilized before an OSD is 
              considered ``full``.

:Type: Float
:Default: ``0.95``


``mon_osd_backfillfull_ratio``

:Description: The threshold percentage of device space utilized before an OSD is
              considered too ``full`` to backfill.

:Type: Float
:Default: ``0.90``


``mon_osd_nearfull_ratio`` 

:Description: The threshold percentage of device space used before an OSD is 
              considered ``nearfull``.

:Type: Float
:Default: ``0.85``


.. tip:: If some OSDs are nearfull, but others have plenty of capacity, you 
         may have an inaccurate CRUSH weight set for the nearfull OSDs.

.. tip:: These settings only apply during cluster creation. Afterwards they need
         to be changed in the OSDMap using ``ceph osd set-nearfull-ratio`` and
         ``ceph osd set-full-ratio``

.. index:: heartbeat

Heartbeat
---------

Ceph monitors know about the cluster by requiring reports from each OSD, and by
receiving reports from OSDs about the status of their neighboring OSDs. Ceph
provides reasonable default settings for monitor/OSD interaction; however,  you
may modify them as needed. See `Monitor/OSD Interaction`_ for details.


.. index:: Ceph Monitor; leader, Ceph Monitor; provider, Ceph Monitor; requester, Ceph Monitor; synchronization

Monitor Store Synchronization
-----------------------------

When you run a production cluster with multiple monitors (recommended), each
monitor checks to see if a neighboring monitor has a more recent version of the
cluster map (e.g., a map in a neighboring monitor with one or more epoch numbers
higher than the most current epoch in the map of the instant monitor).
Periodically, one monitor in the cluster may fall behind the other monitors to
the point where it must leave the quorum, synchronize to retrieve the most
current information about the cluster, and then rejoin the quorum. For the
purposes of synchronization, monitors may assume one of three roles: 

#. **Leader**: The `Leader` is the first monitor to achieve the most recent
   Paxos version of the cluster map.

#. **Provider**: The `Provider` is a monitor that has the most recent version
   of the cluster map, but wasn't the first to achieve the most recent version.

#. **Requester:** A `Requester` is a monitor that has fallen behind the leader
   and must synchronize in order to retrieve the most recent information about
   the cluster before it can rejoin the quorum.

These roles enable a leader to delegate synchronization duties to a provider,
which prevents synchronization requests from overloading the leader--improving
performance. In the following diagram, the requester has learned that it has
fallen behind the other monitors. The requester asks the leader to synchronize,
and the leader tells the requester to synchronize with a provider.


.. ditaa::
           +-----------+          +---------+          +----------+
           | Requester |          | Leader  |          | Provider |
           +-----------+          +---------+          +----------+
                  |                    |                     |
                  |                    |                     |
                  | Ask to Synchronize |                     |
                  |------------------->|                     |
                  |                    |                     |
                  |<-------------------|                     |
                  | Tell Requester to  |                     |
                  | Sync with Provider |                     |
                  |                    |                     |
                  |               Synchronize                |
                  |--------------------+-------------------->|
                  |                    |                     |
                  |<-------------------+---------------------|
                  |        Send Chunk to Requester           |
                  |         (repeat as necessary)            |
                  |    Requester Acks Chuck to Provider      |
                  |--------------------+-------------------->|
                  |                    |
                  |   Sync Complete    |
                  |    Notification    |
                  |------------------->|
                  |                    |
                  |<-------------------|
                  |        Ack         |
                  |                    |


Synchronization always occurs when a new monitor joins the cluster. During
runtime operations, monitors may receive updates to the cluster map at different
times. This means the leader and provider roles may migrate from one monitor to
another. If this happens while synchronizing (e.g., a provider falls behind the
leader), the provider can terminate synchronization with a requester.

Once synchronization is complete, Ceph performs trimming across the cluster. 
Trimming requires that the placement groups are ``active+clean``.


.. confval:: mon_sync_timeout
.. confval:: mon_sync_max_payload_size
.. confval:: paxos_max_join_drift
.. confval:: paxos_stash_full_interval
.. confval:: paxos_propose_interval
.. confval:: paxos_min
.. confval:: paxos_min_wait
.. confval:: paxos_trim_min
.. confval:: paxos_trim_max
.. confval:: paxos_service_trim_min
.. confval:: paxos_service_trim_max
.. confval:: paxos_service_trim_max_multiplier
.. confval:: mon_mds_force_trim_to
.. confval:: mon_osd_force_trim_to
.. confval:: mon_osd_cache_size
.. confval:: mon_election_timeout
.. confval:: mon_lease
.. confval:: mon_lease_renew_interval_factor
.. confval:: mon_lease_ack_timeout_factor
.. confval:: mon_accept_timeout_factor
.. confval:: mon_min_osdmap_epochs
.. confval:: mon_max_log_epochs


.. index:: Ceph Monitor; clock

Clock
-----

Ceph daemons pass critical messages to each other, which must be processed
before daemons reach a timeout threshold. If the clocks in Ceph monitors
are not synchronized, it can lead to a number of anomalies. For example:

- Daemons ignoring received messages (e.g., timestamps outdated)
- Timeouts triggered too soon/late when a message wasn't received in time.

See `Monitor Store Synchronization`_ for details.


.. tip:: You must configure NTP or PTP daemons on your Ceph monitor hosts to 
         ensure that the monitor cluster operates with synchronized clocks.
         It can be advantageous to have monitor hosts sync with each other
         as well as with multiple quality upstream time sources.

Clock drift may still be noticeable with NTP even though the discrepancy is not
yet harmful. Ceph's clock drift / clock skew warnings may get triggered even 
though NTP maintains a reasonable level of synchronization. Increasing your 
clock drift may be tolerable under such circumstances; however, a number of 
factors such as workload, network latency, configuring overrides to default 
timeouts and the `Monitor Store Synchronization`_ settings may influence 
the level of acceptable clock drift without compromising Paxos guarantees.

Ceph provides the following tunable options to allow you to find 
acceptable values.


``mon_tick_interval`` 

:Description: A monitor's tick interval in seconds. 
:Type: 32-bit Integer
:Default: ``5`` 


``mon_clock_drift_allowed`` 

:Description: The clock drift in seconds allowed between monitors.
:Type: Float
:Default: ``0.05``


``mon_clock_drift_warn_backoff`` 

:Description: Exponential backoff for clock drift warnings
:Type: Float
:Default: ``5.00``


``mon_timecheck_interval``

:Description: The time check interval (clock drift check) in seconds 
              for the Leader.

:Type: Float
:Default: ``300.00``


``mon_timecheck_skew_interval``

:Description: The time check interval (clock drift check) in seconds when in
              presence of a skew in seconds for the Leader.

:Type: Float
:Default: ``30.00``


Client
------

``mon_client_hunt_interval``

:Description: The client will try a new monitor every ``N`` seconds until it
              establishes a connection.
              
:Type: Double
:Default: ``3.00``


``mon_client_ping_interval``

:Description: The client will ping the monitor every ``N`` seconds.
:Type: Double
:Default: ``10.00``


``mon_client_max_log_entries_per_message``

:Description: The maximum number of log entries a monitor will generate 
              per client message.

:Type: Integer
:Default: ``1000``


``mon_client_bytes``

:Description: The amount of client message data allowed in memory (in bytes).
:Type: 64-bit Integer Unsigned
:Default: ``100ul << 20``

.. _pool-settings:

Pool settings
=============

Since version v0.94 there is support for pool flags which allow or disallow changes to be made to pools.
Monitors can also disallow removal of pools if appropriately configured. The inconvenience of this guardrail
is far outweighed by the number of accidental pool (and thus data) deletions it prevents.

``mon_allow_pool_delete``

:Description: Should monitors allow pools to be removed, regardless of what the pool flags say?

:Type: Boolean
:Default: ``false``


``osd_pool_default_ec_fast_read``

:Description: Whether to turn on fast read on the pool or not. It will be used as
              the default setting of newly created erasure coded pools if ``fast_read``
              is not specified at create time.

:Type: Boolean
:Default: ``false``


``osd_pool_default_flag_hashpspool``

:Description: Set the hashpspool flag on new pools
:Type: Boolean
:Default: ``true``


``osd_pool_default_flag_nodelete``

:Description: Set the ``nodelete`` flag on new pools, which prevents pool removal.
:Type: Boolean
:Default: ``false``


``osd_pool_default_flag_nopgchange``

:Description: Set the ``nopgchange`` flag on new pools. Does not allow the number of PGs to be changed.
:Type: Boolean
:Default: ``false``


``osd_pool_default_flag_nosizechange``

:Description: Set the ``nosizechange`` flag on new pools. Does not allow the ``size`` to be changed.
:Type: Boolean
:Default: ``false``

For more information about the pool flags see `Pool values`_.

Miscellaneous
=============

``mon_max_osd``

:Description: The maximum number of OSDs allowed in the cluster.
:Type: 32-bit Integer
:Default: ``10000``


``mon_globalid_prealloc`` 

:Description: The number of global IDs to pre-allocate for clients and daemons in the cluster.
:Type: 32-bit Integer
:Default: ``10000``


``mon_subscribe_interval`` 

:Description: The refresh interval (in seconds) for subscriptions. The 
              subscription mechanism enables obtaining cluster maps 
              and log information.

:Type: Double
:Default: ``86400.00`` 


``mon_stat_smooth_intervals``

:Description: Ceph will smooth statistics over the last ``N`` PG maps.
:Type: Integer
:Default: ``6``


``mon_probe_timeout`` 

:Description: Number of seconds the monitor will wait to find peers before bootstrapping.
:Type: Double
:Default: ``2.00``


``mon_daemon_bytes``

:Description: The message memory cap for metadata server and OSD messages (in bytes).
:Type: 64-bit Integer Unsigned
:Default: ``400ul << 20``


``mon_max_log_entries_per_event``

:Description: The maximum number of log entries per event. 
:Type: Integer
:Default: ``4096``


``mon_osd_prime_pg_temp``

:Description: Enables or disables priming the PGMap with the previous OSDs when an ``out``
              OSD comes back into the cluster. With the ``true`` setting, clients
              will continue to use the previous OSDs until the newly ``in`` OSDs for
              a PG have peered.

:Type: Boolean
:Default: ``true``


``mon_osd_prime pg temp max time``

:Description: How much time in seconds the monitor should spend trying to prime the
              PGMap when an out OSD comes back into the cluster.

:Type: Float
:Default: ``0.50``


``mon_osd_prime_pg_temp_max_time_estimate``

:Description: Maximum estimate of time spent on each PG before we prime all PGs
              in parallel.

:Type: Float
:Default: ``0.25``


``mon_mds_skip_sanity``

:Description: Skip safety assertions on FSMap (in case of bugs where we want to
              continue anyway). Monitor terminates if the FSMap sanity check
              fails, but we can disable it by enabling this option.

:Type: Boolean
:Default: ``False``


``mon_max_mdsmap_epochs``

:Description: The maximum number of mdsmap epochs to trim during a single proposal.
:Type: Integer
:Default: ``500``


``mon_config_key_max_entry_size``

:Description: The maximum size of config-key entry (in bytes)
:Type: Integer
:Default: ``65536``


``mon_scrub_interval``

:Description: How often the monitor scrubs its store by comparing
              the stored checksums with the computed ones for all stored
              keys. (0 disables it. dangerous, use with care)

:Type: Seconds
:Default: ``1 day``


``mon_scrub_max_keys``

:Description: The maximum number of keys to scrub each time.
:Type: Integer
:Default: ``100``


``mon_compact_on_start``

:Description: Compact the database used as Ceph Monitor store on
              ``ceph-mon`` start. A manual compaction helps to shrink the
              monitor database and improve the performance of it if the regular
              compaction fails to work.

:Type: Boolean
:Default: ``False``


``mon_compact_on_bootstrap``

:Description: Compact the database used as Ceph Monitor store
              on bootstrap. Monitors probe each other to establish
              a quorum after bootstrap. If a monitor times out before joining the
              quorum, it will start over and bootstrap again.

:Type: Boolean
:Default: ``False``


``mon_compact_on_trim``

:Description: Compact a certain prefix (including paxos) when we trim its old states.
:Type: Boolean
:Default: ``True``


``mon_cpu_threads``

:Description: Number of threads for performing CPU intensive work on monitor.
:Type: Integer
:Default: ``4``


``mon_osd_mapping_pgs_per_chunk``

:Description: We calculate the mapping from placement group to OSDs in chunks.
              This option specifies the number of placement groups per chunk.

:Type: Integer
:Default: ``4096``


``mon_session_timeout``

:Description: Monitor will terminate inactive sessions stay idle over this
              time limit.

:Type: Integer
:Default: ``300``


``mon_osd_cache_size_min``

:Description: The minimum amount of bytes to be kept mapped in memory for osd
               monitor caches.

:Type: 64-bit Integer
:Default: ``134217728``


``mon_memory_target``

:Description: The amount of bytes pertaining to OSD monitor caches and KV cache
              to be kept mapped in memory with cache auto-tuning enabled.

:Type: 64-bit Integer
:Default: ``2147483648``


``mon_memory_autotune``

:Description: Autotune the cache memory used for OSD monitors and KV
              database.

:Type: Boolean
:Default: ``True``


.. _Paxos: https://en.wikipedia.org/wiki/Paxos_(computer_science)
.. _Monitor Keyrings: ../../../dev/mon-bootstrap#secret-keys
.. _Ceph configuration file: ../ceph-conf/#monitors
.. _Network Configuration Reference: ../network-config-ref
.. _Monitor lookup through DNS: ../mon-lookup-dns
.. _ACID: https://en.wikipedia.org/wiki/ACID
.. _Adding/Removing a Monitor: ../../operations/add-or-rm-mons
.. _Monitoring a Cluster: ../../operations/monitoring
.. _Monitoring OSDs and PGs: ../../operations/monitoring-osd-pg
.. _Bootstrapping a Monitor: ../../../dev/mon-bootstrap
.. _Changing a Monitor's IP Address: ../../operations/add-or-rm-mons#changing-a-monitor-s-ip-address
.. _Monitor/OSD Interaction: ../mon-osd-interaction
.. _Scalability and High Availability: ../../../architecture#scalability-and-high-availability
.. _Pool values: ../../operations/pools/#set-pool-values
