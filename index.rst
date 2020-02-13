..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Propose two implementation phases for the LDF EFD, phase 1 is focused on data replication from the Summit EDF to LDF, and phase 2 is focused on aggregation and long-term storage options.

TL;DR
=====

Action items for phase 1.

NCSA:

- Make sure the LSP cluster has enough resources for phase 1. See section 2.1

- Make sure we have mount points on each node to configure and deploy the ``local-path-provisioner``. See section 2.1.1

- Configure DNS for the services listed in section 2.1.2.

- Create and install TLS certificate for the services listed in section 2.1.2.

- Configure firewall to give external access to LDF EFD services from the Tucson network (IP ranges ``140.252.32.0/23`` and ``140.252.33.0/23``) and from the NOAO VPN (IP ranges ``140.252.90.0/23``)


SQuaRE:

- Deploy the LDF EFD using Argo CD.

- Contact IT to whitelist the LSP cluster to connect to the Kafka broker running at the Summit.

- Deploy and test the Replicator connector.

- Investigate DM-23307 to avoid exporting NFS with the ``no_root_squash option``.

- Figure out license with Confluent to run the Replicator connector after the trial period.


Introduction
============

The goal for LDF EFD is to replicate data from the Summit EFD to LDF for long term storage and give users of the LSP access to the data. That also provides a natural backup of the Summit EFD data, and redundancy of some services running at the Summit like InfluxDB and Chronograf.

In :sqr:`034` :cite:`SQR-034` we discuss the replication and the long-term storage options for the LDF EFD. This document proposes a plan for its implementation.

To get it going, we don't need to solve all the aspects of the LDF EFD at once. Thus we propose two implementation phases. Phase 1 is focused on data replication from the Summit EDF to LDF.  In phase 2, we focus on aggregation and long-term storage options.

Phase 1: Data replication
=========================

The most immediate need is to replicate the Summit EFD data to LDF.  The `LSST EFD Client <https://efd-client.lsst.io/>`_ is currently used to access the EFD data from the LSP. It implements helper classes to access InfluxDB.

The retention period for the Summit EFD is one month. For phase 1, we need to define a **mid-term retention period** for InfluxDB until we conclude phase 2.

To guide storage specifications in phase 1, let's define a mid-term retention period for InfluxDB of 6 months. That would guarantee we can replicate the Summit EFD data to InfluxDB at LDF until we decide on the long-term storage options for the LDF EFD.

The minimum LDF EFD deployment to make this possible includes the following services:

- **Argo CD:** used to manage the LDF EFD deployment.

- **Confluent Kafka:**  Kafka, Zookeeper, Kafka connect, and Schema Registry deployments.

- **Kafka replicator connector:**  the data replication from the Summit EFD to the LDF EFD is done through the Kafka replicator connector, as described in :sqr:`034` :cite:`SQR-034`.

- **InfluxDB:** the storage option at LDF during phase 1.

- **Kafka InfluxDB sink connector:**  used to write Kafka topics into InfluxDB.

- **Chronograf:** it has proven to be very useful for analysis of the EFD data and is complementary to the LSP. Chronograf is also essential for the administration and monitoring of InfluxDB.

That's mostly the "EFD application" deployment and is SQuaRE's responsibility. Note that Kapacitor is not required for the LDF EFD unless it proves useful for monitoring the replication.

Implementation details
----------------------

SQuaRE plans on using the same Kubernetes cluster that hosts LSP stable at NCSA (lets call it the LSP cluster). Does this cluster have enough resources for phase 1?

From :sqr:`029` :cite:`SQR-029` investigations, the proposed hardware configuration for an EFD instance is:

- Node spec (4 CPUs, 15 GB memory)

- Size: 4 nodes

- Total cores: 16 vCPUs

- Total memory: 60 GB

- Mid-term storage for Kafka and InfluxDB (4TB, SSD preferentially)


Storage
^^^^^^^

From OCS-REQ-0048, the mean ingestion rate required for the EFD is 1.9MB/s. At this rate, it would be required 30TB to store the raw EFD data for 6 months. However, for the next months, only a  few subsystems are expected to be operational at the Summit (T&S, private communication). Also, based on the current data volumes observed at the Summit EFD, we estimate that 4TB is enough to store the raw Summit EFD at LDF during phase 1.

The EFD deployment uses the ``local-path-provisioner`` to have a storage class available for dynamic provisioning. With that, we don't need to create persistent volumes manually, and we can use the Confluent Kafka, InfluxDB, and Chronograf Helm charts out of the box since they use dynamic provisioning.

The ``local-path-provisioner`` deployment is SQuaRE responsibility, but we expect mount points on each node of the LSP cluster to be ready to use. If these mount points are exported via NFS, we should consider resolving `DM-23307 <https://jira.lsstcorp.org/browse/DM-23307>`_ to avoid exporting NFS with the ``no_root_squash`` option.


Networking
^^^^^^^^^^

Following :sqr:`034` :cite:`SQR-034`, the Kafka replicator connector is deployed at the LSP cluster (destination cluster). That means the Summit network configuration needs to whitelist the LSP cluster to connect to the Kafka broker running at the Summit.

Users of the LSP will access the LDF EFD running on the same cluster, but external access to the LDF EFD services from the Tucson network (IP ranges ``140.252.32.0/23`` and ``140.252.33.0/23``) and the NOAO VPN (IP ranges ``140.252.90.0/23``) is also required.

To enable external access to Chronograf, InfluxDB, Kafka Schema Registry, and Argo CD, the NCSA network configuration needs to whitelist the Ingress IP address.

The LDF EFD Kafka broker does not require external access, it is used by Kafka Connect internally.

We plan on using the same Argo CD already deployed on the LSP cluster. It would be nice to have a separate URL to access this service in addition to the current one.

Given the above, NCSA will manage the DNS configuration and provide a TLS certificate for the following URLs:

* Argo CD: https://lsst-argocd-ldf.ncsa.illinois.edu
* Kafka Schema Registry: https://lsst-schema-registry-ldf-efd.ncsa.illinois.edu
* InfluxDB: https://lsst-influxdb-nts-ldf.ncsa.illinois.edu
* Chronograf: https://lsst-chronograf-ldf-efd.ncsa.illinois.edu

The Ingress controller deployment on the LSP cluster is also managed by NCSA, while the ingress configuration for the LDF EFD services is SQuaRE's responsibility.

The access to these services can be tested even before starting the LDF EFD deployment. We should be able to access Argo CD from the above URL,  and the other services should return a ``404`` from the Ingress controller but show a valid TLS certificate. The access should be tested from NCSA, from Tucson, and using the NOAO VPN.

Authentication
^^^^^^^^^^^^^^

For Chronograf and Argo CD, we plan on using GitHub OAuth with access restricted to the ``lsst-sqre`` GitHub organization, that's the same method currently used by the other EFD instances.

Phase 2: Data Aggregation and long-term storage options
=======================================================

We have a top-level OSS requirement to store ten years of EFD data at LDF or the Base Facility. We need a definition for that before implementing phase 2.

The following is an incomplete list of tasks for the moment. Tater we'll give the implementation details for this phase.

- Develop the aggregator component of the EFD described in :sqr:`034` :cite:`SQR-034`
- Investigate options to store/access data from InfluxDB for a period of 10 years, including downsampling and data roll-up.
- Specify requirements for the Oracle DB (connection details,  permissions, namespaces, table partitioning,  etc)
- Deploy the Kafka Oracle Sink connector. We test this connector in `DM-19655 <https://jira.lsstcorp.org/browse/DM-19655>`_.
- Investigate and test a Kafka connector to write data to Parquet Files.
- Extend the `LSST EFD Client <https://efd-client.lsst.io/>`_ to access data from the Oracle database and Parquet files if needed.

During phase 1 SQuaRE expects to learn more about storage needs for phase 2, as well as computing needs, to query the InfluxDB instance at LDF based on the increasing data volumes accumulated by the EFD.


.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
  :style: lsst_aa
