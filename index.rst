:tocdepth: 1

.. Metadata such as the title, authors, and description are set in metadata.yaml

Abstract
========

We examine the practicality of switching from DDS to Kafka for telescope control commands, events, and telemetry.

Introduction
============

The Rubin Observatory control software uses ADLink OpenSplice DDS to write commands to controllers and read command acknowledgements, events and telemetry messages from those controllers.
Communication uses two in-house designed packages: ts_salobj for Python-based software and ts_sal for C++, Java and LabVIEW.
Rubin Observatory uses Service Abstraction Layer (SAL), an in-house standard, to describe the schema of our messages and the basics of reading and writing messages.
Controllers that receive SAL commands and write SAL events and telemetry messages are called Commandable SAL Components (CSCs).

ADLink is no longer developing OpenSplice DDS; they are switching to a new implementation.
ADLink claims that they will continue to support OpenSplice, but they have not provided a definitive time horizon for that support.
Furthermore, ADLink has already withdrawn the free “community” edition of OpenSplice, which our developers use.

Thus we have started to think about migrating our control software.
SAL was designed to be abstract enough to use other messaging systems, so we are not required to stay with DDS.

Kafka is an appealing alternative to DDS because we already use Kafka widely in the project, including to save SAL messages to the Engineering Facilities Database.
We know Kafka has the necessary throughput, and we have in-house expertise.
Furthermore Kafka is open source, scalable, and widely used.

Our primary concerns are:

* Latency: does Kafka have low enough latency to be used for sending control system commands?

* Compatibility: how difficult would it be to switch our control software to use Kafka?

In order to better evaluate the feasibility of using Kafka for our control system, we created a version of ts_salobj that uses Kafka for communication.
We then used it to measure performance and converted several of our packages to use it.

Similarities Between DDS and Kafka
==================================

Both DDS and Kafka use a publish-subscribe model to transmit and receive messages.
Each message is associated with a particular "topic" that is described by an associated schema.
Messages are sent using TCP/IP.

Both DDS and Kafka support optionally retrieving historical (also called "late-joiner") data, and our control system architecture requires this.
As a node starts up, it reads the most recent historical message for each event it subscribes to, in order to get the current state of the system without having to issue commands to CSCs.
However, we do not read historical data for commands, because reponding to stale commands is dangerous.
We also do not read historical telemetry because telemetry is output frequently, so there is no need, and we want to be sure it the data is current.

Differences Between DDS and Kafka
=================================

Kafka is a broker-based system, whereas DDS is distributed, with no central broker.

With DDS each node (e.g. CSC or Jupyter Notebook) discovers the other nodes, using UDP multicast, as the software starts up.
Reliability is achieved by switching which node acts as the primary source of data if one fails.
DDS (using our current configuration) publishes topic data using multicast, which means each message is only sent over the network once.

With Kafka each node connects to a specified broker and schema registry; there is no discovery phase and no need for UDP multicast.
Kafka writers write messages to the broker, and readers read messages from the broker.
This is a much simpler topology than DDS, but it does require that each message be sent to the broker and then again to each subscriber.
Reliability and performance can be increased by running multiple brokers in parallel, and by using more than one partition for each topic.

Schema Evolution
----------------

DDS does not support schema evolution, at least as we are using it.
Thus, in order to prevent collisions between old topic schemas and new topic schemas, the DDS topic name includes a hash that we change whenever the schema changes.
Our high-level software hides that hash from users.

Kafka schemas support basic schema evolution: we may add or delete fields and readers and writers can freely use older or newer schemas without restarting.
Furthermore, even if a schema is not compatible for some topics, all we need to do is restart the publishers and subscribers for those topics to use the new schema; we do not have to bring down the whole system, as we do for DDS.

Licensing
---------

With ADLink OpenSplice we presently run two very different versions of the software:

* Developers use the free community edition of OpenSplice, in order to reduce our costs and to avoid licensing hassles.
  ADLink no longer offers a free edition of OpenSplice, and it only ever supported single-process topic daemons, which cannot handle the demands of our full control system.
  The free edition was intentionally kept at least one major version behind the licensed version; it was withdrawn at version 6.9, while the licensed software is at 6.11.
  The free edition was poorly maintained before ADLink withdrew it; several bug-fix pull requests we submitted were never merged.

* Deployment uses the licensed version of OpenSplice.
  We do this primarily because it supports shared-memory topic daemons, which can handle the demands of our full system.
  The commerical version is also required to run ADLink's diagnostic tools.

Kafka is open-source.
Thus we can use the same version for development and deployment.
This also eliminates the need to maintain licenses.

Reliability
-----------

We have run into several serious bugs in ADLink OpenSplice over the years, both when we first wrote ts_sal and ts_salobj, and then again as we attempted to upgrade to new releases of OpenSplice.
ADLink helpfully fixed some of these bugs, gave us workarounds for others, and we discovered some workarounds ourselves.
Several of these workarounds are still in our code today.

We are still using licensed OpenSplice 6.10, instead of 6.11, because both 6.11.0 and 6.11.1 had serious bugs.
It costs us several people-days of effort to test each release.
If we run into problems, we need to spend additional time trying to isolate the bugs, so we can report them.
We have not yet tried 6.11.2, and we are not sure if it is worth the effort.

When we wrote the Kafka version of ts_salobj we did not run into any bugs at all.

Kafka includes an extensive suite of unit tests and integration tests.
This `posting <https://www.confluent.io/blog/apache-kafka-tested/>`_ has more information.

Support
-------

We pay ADLink for OpenSplice DDS support.
In exchange we get fixes and workarounds for bugs that we report, as well as the right to use the commecial version for deployment.

We are unlikely to need to pay for Kafka support.
We already have extensive in-house Kafka expertise, since we use it for populating the Engineering Facilities Database, and for producing alerts.
In addition, Kafka has a large user community, good documentation, and copious online resources (much more so than ADLink OpenSplice DDS).

The Engineering Facilities Database
-----------------------------------

DDS requires running complex extra software (ts_salkafka, an in-house package) in order to copy all messages from DDS to Kafka, so we can ingest them into the Engineering Facilities Database.
If we use Kafka then we can eliminate this extra software.

Subtle Differences Between DDS and Kafka
----------------------------------------
DDS topics have the concept of “quality of service” (QoS) settings for topics. There are many QoS settings, and the settings must match quite closely between publishers and subscribers of a particular topic, else the topic cannot be constructed.
This has been a source of much frustration over the years, though we have finally found QoS settings that work.

The settings for publishing Kafka topics are far simpler than for DDS, and subscribers have no settings.
Kafka publishers can specify the number of partitions (increasing the number of partitions increases throughput) and how many acks to listen for when writing a message (a tradeoff between latency and reliability).

DDS topics have the concept of “aliveness” (one of the many QoS settings), whereas Kafka topics do not.
We have configured our DDS topics such that messages are no longer alive when a CSC exits, in order to avoid collisions between older and newer versions of topic QoS when upgrading control software.
Changes to a topic's schema are handled by changing a hash field in the DDS topic's name, as mentioned in the schema evolution section.
The result is a completely different topic, as far as DDS is concerned.
Until we learned to do this, we frequently ran into a problem where we could not run a new version of a CSC, because it could not create its topics due to an incompability with cached "zombie" data.
But marking topics as dead when a CSC exits means that historical data for CSCs that have quit is not unavailable from DDS.

Kafka has no concept akin to "aliveness", but Kafka doesn't need it.
In most cases we can simply run the new software.
If a topic's schema has changed in some incompatible way then we will have to restart the broker and schema registry.

Performance
===========

Our primary requirements are as follows:

* We must be able to write tracking commands at 20 Hz with a latency of better than 25 ms, with standard deviation of 3 ms.
  This flows down from pointing component requirements LTS-TCS-PTG-0008 and LTS-TCS-PTG-0001 in LTS-583, which require lead-times between 50-70ms with standard deviation of 3ms for tracking demands.

* We must be able to write large telemetry topics, such as MTM1M3 forceActuatorData, at 50 Hz.
  
* All readers must be able to keep up with the data they read.

* The system must be able to handle the traffic of all controllers running at the same time.
  This includes the software that copies messages into the Engineering Facilities Database.

We already know that DDS and Kafka can both keep up with a fully loaded system, because our current system uses DDS, but copies every message to Kafka for ingestion into the Engineering Facilities Database.
We have tested a fully loaded system on a test stand and on the summit.

Thus for measuring performance we concentrated on the first three items.

We tested performance for three SAL topics:

* ``MTM1M3`` ``summaryState`` event: one of our smallest topics.

* ``MTMount`` ``trackTarget`` command: the command for which we most care about latency.

* ``MTM1M3 ``forceActuatorData`` telemetry: one of our largest topics, and one that is written at our highest data rate of 50 Hz.

For Kafka we ran the tests for using two configurations:

* The writer does not wait for acknowledgement (acks = 0).
  This is not considered safe, and is provided purely to show how much latency is due to waiting for acknowledgement.

* The writer waits for one acknowledgement (acks = 1).
  This is a very common configuration that is considered safe.

For DDS we used our standard OpenSplice single-process configuration `ospl-std.xml <https://github.com/lsst-ts/ts_ddsconfig/blob/develop/config/ospl-std.xml>`_ and `QoS <https://github.com/lsst-ts/ts_ddsconfig/blob/develop/qos/QoS.xml>`_.
Note that each topic category (command, event, and telemetry) has its own QoS.

All timings were measured on summit computer ``azar02`` using the following Docker images that communicated with each other:

* An lsstts/develop-env Docker image running the reader and writer.
  This is the only image needed for the DDS timing tests.

* Three Confluent Kafka images: one broker, one schema registry, and one zookeeper.
  These images were started and stopped using ``docker-compose`` with the ``docker-compose.yaml`` file in `kafka-aggregator <https://github.com/lsst-sqre/kafka-aggregator>`.
  Note that these Docker images are free.

See each section for more details.

See also `Apache Kafka Performance <https://developer.confluent.io/learn/kafka-performance/>`_ for other performance measurements.


Latency
-------

Latency was determined by writing 2000 messages at 20 Hz and measuring the time between when each message was written and when the reading process received that message.

+---------+-------+---------------------+-------------------------+
| System  | Acks  | Topic               | Latency (ms)            |
|         |       |                     |                         |
+---------+-------+---------------------+------+------+-----+-----+
|         |       |                     | mean | sdev | min | max |
+=========+=======+=====================+======+======+=====+=====+
| DDS     |  n/a  |  summaryState       |   1  |   0  |   1 |   1 |
+---------+-------+---------------------+------+------+-----+-----+
| DDS     |  n/a  |  trackTarget        |   1  |   0  |   1 |   2 |
+---------+-------+---------------------+------+------+-----+-----+
| DDS     |  n/a  |  forceActuatorData  |   5  |   0  |   4 |   7 |
+---------+-------+---------------------+------+------+-----+-----+
| Kafka   |  0    |  summaryState       |   2  |   0  |   2 |   6 |
+---------+-------+---------------------+------+------+-----+-----+
| Kafka   |  0    |  trackTarget        |   2  |   0  |   2 |   4 |
+---------+-------+---------------------+------+------+-----+-----+
| Kafka   |  0    |  forceActuatorData  |   3  |   0  |   3 |   8 |
+---------+-------+---------------------+------+------+-----+-----+
| Kafka   |  1    |  summaryState       |   2  |   1  |   2 |  27 |
+---------+-------+---------------------+------+------+-----+-----+
| Kafka   |  1    |  trackTarget        |   2  |   1  |   2 |  25 |
+---------+-------+---------------------+------+------+-----+-----+
| Kafka   |  1    |  forceActuatorData  |   3  |   0  |   3 |  23 |
+---------+-------+---------------------+------+------+-----+-----+

Kafka maximum latency is is on the edge of our requirements with acks=1,
though the mean and standard deviations are small; indicating that Kafka has occasional outliers.
Options for handling this include:

* Send tracking commands 10-20ms farther in advance.
  This will decrease the responsiveness to new slews by the same small amount.

* Live with it.
  Occasional tracking commands will have a few ms of extrapolation.
  Our control system is designed to handle this.

At one time Kafka had a reputation for long latency.
We suspect this has improved because of the high-performance `Azul JVM <https://www.azul.com/products/core>`_ which Confluent uses in its Kafka docker images.

Write Speed
-----------

Write speed was measured by writing 10,000 messages (with one exception, noted below) as quickly as ts_salobj could write them.
We also report read speed, but please note that it is not a measure of maximum read speed.
As long as the reader can keep up with the writer, read speed should be approximately the same as write speed.

+--------+------+-------------------+--------------+--------------+----------+
| System | Acks | Topic             | Write Speed  | Read Speed   | Lost     |
|        |      |                   |              |              | Messages |
+--------+------+-------------------+--------------+--------------+----------+
|        |      |                   | messages/s   | messages/s   |          |
+========+======+===================+==============+==============+==========+
| DDS    | n/a  | summaryState      | 16,553       | 16,580       |      0   |
+--------+------+-------------------+--------------+--------------+----------+
| DDS    | n/a  | trackTarget       | 13,557       | 13,577       |      0   |
+--------+------+-------------------+--------------+--------------+----------+
| DDS    | n/a  | forceActuatorData |  1,912       |  1,492       |  2,632   |
+--------+------+-------------------+--------------+--------------+----------+
| Kafka  | 0    | summaryState      |  4,730       |  4,739       |      0   |
+--------+------+-------------------+--------------+--------------+----------+
| Kafka  | 0    | trackTarget       |  4,379       |  4,385       |      0   |
+--------+------+-------------------+--------------+--------------+----------+
| Kafka  | 0    | forceActuatorData |  3,267       |  3,262       |      0   |
+--------+------+-------------------+--------------+--------------+----------+
| Kafka  | 1    | summaryState      |  2,065       |  2,066       |      0   |
+--------+------+-------------------+--------------+--------------+----------+
| Kafka  | 1    | trackTarget       |  1,841       |  1,842       |      0   |
+--------+------+-------------------+--------------+--------------+----------+
| Kafka  | 1    | forceActuatorData |  1,652       |  1,652       |      0   |
+--------+------+-------------------+--------------+--------------+----------+

These throughputs all easily meet our requirements.

The fact that some DDS messages were lost simply shows that DDS salobj can write large messages faster than it can read them.
This is not a cause for concern, as long as we keep up with the actual rate at which messages are written.
In practice, our deployed control system does not presently lose messages.

In order for the DDS forceActuatorData read test to finish, despite losing data, we wrote 20,000 messages for that test.
Any number large enough that the reader can read 10,000 messages should give the same results;
the reader quits as soon as it has seen the specified number of messages, while the writer can keep going.

We are not sure why reported read speed is sometimes slightly higher than write speed.

Measurement Details
-------------------

* The DDS tests used ts_salobj v7.0.0.

* The Kafka tests used the ``kafka`` branch of ts_salobj (commit d2ef9a4)

* The tests used ``measure_read_speed.py`` and ``measure_write_speed.py`` from the ``kafka`` branch.

Unit Test Performance
---------------------

Unit tests run much faster using Kafka than with DDS, probably because the single-process version of OpenSplice DDS is very slow to start.

On ``azar02`` Kafka ts_salobj unit tests run in 7 minutes whereas DDS unit tests run in 17 minutes.
We observe similar speedups for the other packages we converted.

Converting to Kafka
===================

To convert our control system to Kafka we would need to do the following:

* Convert ts_salobj to use Kafka.
  This has already been done.

* Update Python CSCs.
  This is discussed below.
  We demonstrate that many packages are already compatible, and the others we have tried need few changes.

* Convert ts_sal to use Kafka.
  This is discussed below.

* Update non-python CSCs.
  This is discussed below.

* Use the Kafka broker that feeds the EFD as our main broker.

Converting Python CSCs
-----------------------

Packages that use ts_salobj should be largely compatible with the Kafka version of ts_salobj.
This section discusses the changes that will be required in the code and build files.

The required code changes include:

#. Code must await ``SalInfo.start()`` before writing data.

   This shows up is in the ``start`` method of many CSCs.
   For historical reasons CSCs usually called ``super().start()`` last, but that is no longer necessary in ts_salobj 7.1.

   In addition, one package (ts_watcher) creates its own ``SalInfo`` instances in unit tests.
   These should changed to call ``await sal_info.start()``.

   These changes are compatible with ts_salobj 7.1 and have been back-ported for cycle 26 for many packages.

* Messages no longer have a ``get_vars`` method.
  
  Call ``vars(message)`` instead of ``message.get_vars()``.
  Only a few packages calls get_vars, including ts_scriptqueue and ts_watcher.

  One may call ``vars`` on DDS messages, but it returns one unwanted extra field: ``_member_attributes``.
  It is a nuisance to ignore that extra field, so we do not recommend back-porting this change.
  
* Explicitly encode bytes arrays for string-valued message fields.

  OpenSplice DDS allows writing a bytes array as a string, encoding the data internally, but Kafka requires the data be explicitly encoded.
  We do not expect to find many instances of this.

  We do not recommend back-porting this change, because OpenSplice DDS does not support ``utf-8``, which is our preferred encoding.

* Unit tests run much faster with Kafka.

  As a result, poorly written tests may fail due to timing issues.

  We strongly recommend back-porting such changes, because they make the tests more robust.

The required build changes are as follows:

* Require Kafka-related packages instead of DDS-related packages.

* Stop building IDL files in Jenkinsfile.
  Kafka ts_salobj parses our ts_xml topic schema files directly.

We tested Kafka ts_salobj compatibility against many CSC packages, after making the first change (await ``SalInfo.start()``) listed above.

The following packages worked with no additional changes:

* ts_atwhitelight
* ts_ATDome
* ts_ATDomeTrajectory
* ts_ATMCSSimulator
* ts_ATPneumaticsSimulator
* ts_mtdome
* ts_mtdometrajectory
* ts_mthexapod
* ts_mtrotator
* ts_salobjATHexapod
* ts_standardscripts
* ts_weatherstation

The following packages required a few additional changes:

ts_mtmount
~~~~~~~~~~

Running the kafka version of ts_salobj exposed two bugs in the MTMount CSC:

* The CSC would not reliably go to FAULT if it lost connection to the low-level controller.

* The mock controller would not reliably publish telemetry on reconnection, due to a bug in ts_tcpip.

These fixes have been back-ported.

ts_observatory_control
~~~~~~~~~~~~~~~~~~~~~~

This package uses topic metadata to generate mocks for unit tests; that required some simple changes.

ts_scriptqueue
~~~~~~~~~~~~~~

This package has deep usage of ts_salobj and, unsurprisingly, needed a few changes:

* Change one use of ``message.get_vars()`` to ``vars(message)``.

* Change one instance of writing a bytes array to a string field, by encoding the bytes array.

We do not plan to back-port these trivial changes.

ts_watcher
~~~~~~~~~~

This package has deep usage of ts_salobj and, unsurprisingly, needed a few changes:

* Change one use of ``message.get_vars()`` to ``vars(message)`` in a unit test.
  That change was back-ported, because the data is used in such a way that ``vars`` works with DDS messages.

* One unit test subclasses ``lsst.ts.salobj.topics.WriteTopic`` to write data with an offset value of ``private_sndStamp``.
  This uses some internal details of ``WriteTopic`` (which is not ideal, but we have not found a cleaner solution), and these details are different for the Kafka version of ts_salobj.

Converting ts_sal and C++, Java, and LabVIEW CSCs
-------------------------------------------------

To port ts_sal from DDS to Kafka, the bulk of the effort would likely be in transitioning the topic message schema from OMG IDL to Avro.

We have already tested a version of ts_sal that published Kafka messages, during the MTM1M3 early test campaign.
The MTM1M3 simulator was used to generate the full set of events and telemetry, which were captured by a SAL EFD writer process which published everything to Kafka.
The Kafka brokers kept up with the load, despite the fact that the brokers were in the cloud.
The broker was not using the low-latency Azul JVM, though, and using acks=1 we could see significant latency, as exhibited by the acks arriving in bunches.

For the subscription side, we can run some early tests in our current system by having a SAL Kafka subscriber monitor the existing Kafka messages that feed the Engineering Facilities Database.

We do not expect much change to the SAL API, except for a few specialty calls whose names assume that the underlying transport is DDS, e.g. ``getOSPLVersion()`` would become ``getKafkaVersion()``, or possibly ``getMiddlewareTransportVersion()``.

Kafka is written in Java and there are multiple C++ wrapper options available as well.
The SAL LabVIEW support is layered on top of the C++ API, so minimal impact is expected there.

Application level code impact should therefore be minimal for CSCs based on C++, Java, and LabVIEW.

Developer Impact
================

Developers will have to run their own Kafka server to run unit tests.
This turns out to be trivial using docker-compose.
The kafka version of ts_salobj has the necessary configuration file and instructions in the user guide, which are as follows:

    If running tests in a Docker image, run the image with option ``--network=ts_salobj_default_default``.

    To start Kafka, issue this command in ts_salobj's main directory: ``docker-compose up -d zookeeper broker schema-registry``

    To stop Kafka completely, issue this command in ts_salobj's main directory: ``docker-compose rm --stop --force broker schema-registry zookeeper``.
    If you stop these processes without removing them, they will retain their data, and if you run them too long, this may eventually use up resources.

Conclusions
===========

Kafka performance meets our needs, though latency may require sending tracking commands 10-20 ms earlier than we currently plan.

Kafka offers many advantages over ADLink OpenSplice DDS:

* Better reliability.
* Support for schema evololution, which simplifies deploying new versions of our control software.
* We have extensive in-house Kafka expertise, because we use Kafka to populate the Engineering Facilities Database and to publish science alerts.
* Developers can run the same version of Kafka that we deploy.
* Better documentation, including extensive on-line resources.
* No software licenses.
  This saves money and eliminates the time spent renewing and maintaining licenses.

We have proven that converting our Python code is simple, and much of our code is already compatible.

Appendix: Technical Details of the Kafka Implementation
=======================================================

Schemas
-------

Confluent Schema Registry supports `three schema formats <https://www.confluent.io/blog/confluent-platform-now-supports-protobuf-json-schema-custom-formats/>`_: Avro, Google protobuf, and json schema.

Our code that feeds Kafka data to the Engineering Facilities Database uses Avro, and would not be easy to switch.
So for the Kafka ts_salobj prototype we used Avro.

Nonetheless we explored other options before making this decision, and present our findings here.

Support for Google protobuf, and json schema is recent; Avro is the original Kafka schema.

Here are two comparisons between Avro and protobuf: a `summary <https://blog.softwaremill.com/the-best-serialization-strategy-for-event-sourcing-9321c299632b>`_ that likes both, and a `detailed look <detailed look>`_, including schema evolution.
Google protobuf has similar advantages and disadvantages to Avro.
Therefore we see no strong reason to prefer protobuf over Avro.

json schema is what we use for CSC configuration files, so we have experience with it.
However, information is very sparse about how Kafka uses json schema.
For example: does Kafka fill in default values? The standard json schema validators do not.

Schema Evolution
----------------

Avro was designed with schema evolution in mind.
One may explicitly add or delete fields, so long as they specify an explicit default value.
Readers using an older version of the schema will not see data for added fields, and will get default values for deleted fields.

Google protobuf also was designed with schema evolution in mind.
It uses manually numbered fields, and as long as you don’t re-use a number you can add new fields and delete old ones.

json schema was not designed for schema evolution, but Kafka does support it, as `described here <https://yokota.blog/2021/03/29/understanding-json-schema-compatibility/>`_.
We tested schema evolution and found, as explained in that link, that setting additionalProperties=false allows one to add fields, but not delete them.
Fortunately, setting additionalProperties=false is our preferred way to define schemas, because it detects typos in field names.
However, the inability to delete fields is a drawback.

Arrays
------

SAL arrays are all fixed-length.
This is a significant issue, because neither Avro nor protobuf supports any kind of constraint on array length.
json schema is the only choice that supports array length constraints, and we would have to test if Kafka enforces them.

In order to constrain array lengths using Avro or Google protobuf we will have to manually check the data ourselves.
This is practical as long as we retain additional information (beyond that used to generate the schema used by Kafka) when we parse the XML files that define our SAL interfaces.
The Kafka version of ts_salobj checks array lengths.

Scalar Data Types
-----------------

SAL supports the following scalar field types: boolean, byte/octet (uint8), short (int16), unsigned short (uint16), int/long (int32), unsigned int/long (uint32), long long (int64), unsigned long long (uint64), float, double, string (unicode).
The slashes indicate pairs of names that mean the same thing, e.g. byte and octet both translate to uint8.
Whichever Kafka schema we choose must also support these, using a larger type for integers and double for float, if necessary.
Note that this is true whether or not we use Kafka for T&S communication, because we use Kafka to ingest T&S messages into the Engineering Facilities Database.

Avro supports null, boolean, int (int3232), long (int64), float, double, bytes (sequence of 8-bit unsigned values), and string (unicode).
This supports all the SAL data types except uint64, for which int64 is a poor representation.
Fortunately uint64 fields are only used for a few fields for one SAL component: MTAOS (plus the Test component, which exercises all supported field types), and it is likely we can eliminate that usage.
We must solve this, in order to properly ingest the data into the Engineering Facilities Database.

Protobuf supports supports bool, float, double, int32, int64, uint32, uint64 (and several variants of these with different encoding), float, double, bytes, string.
This includes all of the SAL data types.

json schema supports the following scalar types: null, boolean, integer (we have not found a specification for the size), number (float or integer), string.
Note the weak support for numeric data types: only integer and float.

Default Float Value
-------------------

The default value for float is a tricky question.
NaN is most principled, but it has issues:

* For Avro: NaN is not yet supported as a default in C++, though there is a `patch <https://issues.apache.org/jira/browse/AVRO-2880>`_ that may be accepted.
  If we want to use NaN as the default float value then we must make sure C++, Java, and Python all support it.

* Protobuf has a fixed default value for each type, and for float the default is 0.

* Json schema is like Avro in supporting per-field default values.
  However, we don’t know two details:
  
  * Does Kafka use the specified default values? The standard json schema validators do not.
  
  * Does Kafka allows NaN? Json does not, but extensions often do (and allow it to be spelled as "nan").

We presently use 0 as the default value for floats, because that is the only value ADLink OpenSplice supports.
Thus switching to NaN has the potential to cause problems, though we make little use of default values, so this may not be an issue in practice.
Most likely, our main use of default values will be for schema evolution: any field in the schema being read, but not the schema being written, will have the default value.

Namespaces for Isolation
------------------------

It is important to be able to isolate subsystems from each other, both when running prerelease versions of software on the summit, and for unit testing anywhere.

DDS has two mechanisms for isolation: domain ID and partition names.
Domain ID is more complete isolation: messages using one domain ID cannot see messages on any other domain.
Partition names offer less isolation: topics in all partitions must share the same schemas and quality of service settings.

Kafka is simpler because one can run separate Kafka brokers to isolate systems.
This is especially appropriate for running unit tests, as each package can spin up its own broker.
However we also want to be able to isolate different versions of topics on the main Kafka system, e.g. in order to run an experimental version of a controller.
In order to support this, the Kafka verison of ts_salobj includes a namespace component in each topic name,
where the value is specified by environment variable ``LSST_TOPIC_SUBNAME``.

Python Library
--------------

We implemented the Kafka version of ts_salobj using aiokafka, because it is widely used, has a high-level API, and supports asyncio.
The obvious alternative is confluent_kafka, which is maintained by Confluent, the main company that provides commercial support for Kafka.
Confluent_kafka is well supported and is designed to be fast, but has a rather clumsy API and no support for asyncio.

We also implemented a minimal prototype using confluent_kafka and observed similar performance to aiokafka.
This was most likely due to waiting for 1 ack when writing messages with the prototype; that probably makes the broker the performance limitation, not the client library.

Note that ts_salobj uses confluent_kafka to specify topics, because aiokafka has no admin support.
Thus we need confluent_kafka, regardless of whether we use it for reading and writing messages.

Parsing XML Files
-----------------

The Kafka version of ts_salobj parses ts_xml topic schema files directly on the fly, instead of parsing OMG IDL files as the DDS version does.
This turns out to be very fast; tests/test_speed.py on a mac report: "Created 859.3 topic classes/sec (129 topic classes); total duration 0.15 seconds" when parsing topic classes for MTM1M3, a component which has an unusually large number of topics, including several unusually large topics.

Other Changes
-------------

* Remove the ``private_revCode`` field from topics.
  Kafka has no need for a schema-related revision code.
