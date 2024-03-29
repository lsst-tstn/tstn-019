Helm Charts
===========

The CSC deployment starts with a `Helm <https://helm.sh/>`_ chart.
We are currently adopting version 1.8+ of ArgoCD which works with Helm version 3.
The code for the charts are kept in the `Helm chart GitHub repository <https://github.com/lsst-ts/charts>`_.
The next two sections will discuss each chart in detail.
For a description of the APIs used, consult the `Kubernetes documentation <https://kubernetes.io/docs/reference/>`_ reference.
The chart sections will not go into great detail on the content of each API delivered.
Each chart section will list all of the possible configuration aspects that each chart is delivering, but full use of that configuration is left to the `ArgoCD Configuration` section and examples will be provided there.
For the CSC deployment, we will run a single container per pod on Kubernetes.
The Kafka producers will follow the same pattern.
The OSPL daemon instances will use a init container on the summit to handle routing aspects for the DDS communication.

Cluster Configuration Chart
---------------------------

This chart (`cluster-config`) is responsible for setting up all of the namespaces for a Kubernetes cluster by using Namespace from the Kubernetes Cluster API.
Its only configuration is a list of namespaces that the cluster will use.

.. list-table:: Cluster Configuration Chart YAML Configuration
   :widths: 15 25
   :header-rows: 1

   * - YAML Key
     - Description
   * - namespaces
     - This holds a list of namespaces for the cluster

OSPL Configuration Chart
------------------------

This chart (`ospl-config`) is responsible for ensuring the network interface for OpenSplice DDS communication is set to listen to the proper one on a Kubernetes cluster.
The `multus` CNI provides the multicast interface on the Kubernetes cluster for the pods. The rest of the options deal with configuring the shared memory configuration the control system is using.
The chart uses ConfigMap from the Kubernetes Config and Storage API to provide the `ospl.xml` file for all of the cluster's namespaces.

.. list-table:: OSPL Configuration Chart YAML Configuration
   :widths: 15 25
   :header-rows: 1

   * - YAML Key
     - Description
   * - namespaces
     - This holds a list of namespaces for the cluster
   * - networkInterface
     - The name of the `multus` CNI interface
   * - domainId
     - This sets the domain ID for the DDS communication
   * - shmemSize
     - The size in bytes of the shared memory database
   * - maxSamplesWarnAt
     - The maximum number of samples at which the system warns of resource
       issues
   * - schedulingClass
     - The thread scheduling class that will be used by a daemon service
   * - schedulingPriority
     - The scheduling priority that will be used by a daemon service
   * - monitorStackSize
     - The stack size in bytes for the daemon service
   * - waterMarksWhcHigh
     - This sets the size of the high watermark. Units must be explicitly used
   * - waterMarksWhcHighInit
     - This sets the the initial size of the high watermark. Units must be explicitly used
   * - waterMarksWhcAdaptive
     - This makes the high watermark change according to system pressure
   * - deliveryQueueMaxSamples
     - This controls the maximum size of the delivery queue in samples
   * - dsGracePeriod
     - This sets the discovery time grace period. Time units must be specified
   * - squashParticipants
     - This controls whether one virtual (true) or all (false) domain
       participants as shown at discovery time
   * - namespacePolicyAlignee
     - This determines how the durability service manages the data that matches
       the namespace
   * - domainReportEnabled
     - This enables reporting at the Domain level
   * - ddsi2TracingEnabled
     - This enables tracing for the DDSI2 service
   * - ddsi2TracingVerbosity
     - This sets the level of information for the DDSI2 tracing
   * - ddsi2TracingLogfile
     - This specifies the location and name of the DDSI2 tracing log
   * - durabilityServiceTracingEnabled
     - This enables tracing for the Durability service
   * - durabilityServiceTracingVerbosity
     - This sets the level of information for the Durability tracing
   * - durabilityServiceTracingLogfile
     - This specifies the location and name of the Durability tracing log
   * - durabilityServiceInitialDiscoveryPeriod
     - This sets the time for the initial discovery period during startup. Time in seconds
   * - durabilityServiceAlignmentComboPeriodInitial
     - This sets the maximum amount of time (seconds) a system waits for alignment after
       an alignment request while the system is not operational
   * - durabilityServiceAlignmentComboPeriodOperational
     - This sets the maximum amount of time (seconds) a system waits for alignment after
       an alignment request while the system is operational


OSPL Daemon Chart
-----------------

This chart (`ospl-daemon`) handles deploying the OSPL daemon service for the shared memory configuration.
This daemon takes over the communication startup, handling and tear down from the individual CSC applications.
The chart uses a DaemonSet from the Kubernetes Workload APIs since it is designed to run on every node of a Kubernetes cluster.

.. list-table:: OSPL Daemon Chart YAML Configuration
   :widths: 15 25
   :header-rows: 1

   * - YAML Key
     - Description
   * - image
     - This section holds the configuration of the container image
   * - image.repository
     - The Docker registry name of the container image
   * - image.tag
     - The tag of the container image
   * - image.pullPolicy
     - The policy to apply when pulling an image for deployment
   * - image.nexus3
     - The tag name for the Nexus3 Docker repository secrets if private images
       need to be pulled
   * - namespace
     - This is the namespace in which the CSC will be placed
   * - env
     - This section holds a set of key, value pairs for environmental variables
   * - useExternalConfig
     - This sets whether to rely on the ConfigMap for OSPL configuration or the internal
       OSPL configuration
   * - osplVersion
     - This is the version of the OpenSplice library to run. It is used to set the 
       location of the OSPL configuration file
   * - initContainer
     - This section sets the option use of an init container for DDS route fixing
   * - initContainer.repository
     - The Docker registry name of the init container image
   * - initContainer.tag
     - The tag of the init container image 
   * - initContainer.pullPolicy
     - The policy to apply when pulling an image for init container deployment
   * - shmemDir
     - This is the path to the Kubernetes local store where the shared memory
       database will be written
   * - useHostIpc
     - This sets the use of the host inter-process communication system.
       Defaults to true
   * - useHostPid
     - This sets the use of the host process ID system. Defaults to true
   * - resources
     - This allows the specifications of resources (CPU, memory) requires to run the
       container
   * - nodeSelector
     - This allows the specification of using specific nodes to run the pod
   * - affinity
     - This specifies the scheduling constraints of the pod
   * - tolerations
     - This specifies the tolerations of the pod for any system taints

Kafka Producer Chart
--------------------

While not a true control component, the Kafka producers are nevertheless an important part of the control system landscape.
They have the capability to convert the SAL messages into Kafka messages that are then ingested into the Engineering Facilities Database (EFD). See :cite:`SQR-034` for more details. 

The chart consists of a single Kubernetes Workloads API: Deployment.
The Deployment API allows for restarts if a particular pod dies which assists in keeping the producers up and running all the time.
For each producer specified in the configuration, a deployment will be created. We will now cover the configuration options for the chart.

.. list-table:: Kafka Producer Chart YAML Configuration
   :widths: 15 25
   :header-rows: 1

   * - YAML Key
     - Description
   * - image
     - This section holds the configuration of the container image
   * - image.repository
     - The Docker registry name of the container image to use for the producers
   * - image.tag
     - The tag of the container image to use for the producers
   * - image.pullPolicy
     - The policy to apply when pulling an image for deployment
   * - image.nexus3
     - The tag name for the Nexus3 Docker repository secrets if private images
       need to be pulled
   * - env
     - This section holds environment configuration for the producer container
   * - env.lsstDdsPartitionPrefix
     - The LSST_DDS_PARTITION_PREFIX name applied to all producer containers
   * - env.brokerIp
     - The URI for the Kafka broker that received the generated Kafka messages
   * - env.brokerPort
     - The port associated with the Kafka broker specified in brokerIp
   * - env.registryAddr
     - The URL for the Kafka broker associated schema registry
   * - env.partitions
     - The number of partitions that the producers are supporting
   * - env.replication
     - The number of replications available to the producers
   * - env.waitAck
     - The number of Kafka brokers to wait for an ack from
   * - env.logLevel
     - This value determines the logging level for the producers
   * - env.extras
     - This section holds a set of key, value pairs for environmental variables
   * - producers
     - This section holds the configuration of the individual producers [#]_
   * - producers.name
     - This key gives a name to the producer deployment and can be repeated
   * - producers.name.cscs [#]_
     - The list of CSCs that the named producer will monitor
   * - producers.name.image
     - This section provides optional override of the default image section
   * - producers.name.image.repository
     - The Docker registry container image name to use for the named producer
   * - producers.name.image.tag
     - The container image tag to use for the named producer
   * - producers.name.image.pullPolicy
     - The policy to apply when pulling an image for named producer deployment
   * - producers.name.env
     - This section provides optional override of the defaults env section
   * - producers.name.env.lsstDdsPartitionPrefix
     - The LSST_DDS_PARTITION_PREFIX name applied the named producer container
   * - producers.name.env.partitions
     - The number of partitions that the named producer is supporting
   * - producers.name.env.replication
     - The number of replications available to the named producer
   * - producers.name.env.waitAck
     - The number of Kafka brokers to wait for an ack from for the named
       producer
   * - producers.name.env.logLevel
     - This value determines the logging level for the named producer
   * - producers.name.env.extras
     - This section holds a set of key, value pairs for environmental variables
       for the named producer
   * - namespace
     - This is the namespace in which the producers will be placed
   * - useMulticast
     - This sets the use of the annotation the `multus` address binding needed for DDS
       communication
   * - useExternalConfig
     - This sets whether to rely on the ConfigMap for OSPL configuration or the internal
       OSPL configuration
   * - osplVersion
     - This is the version of the OpenSplice library to run. It is used to set the 
       location of the OSPL configuration file
   * - initContainer
     - This section sets the option use of an init container for DDS route fixing
   * - initContainer.repository
     - The Docker registry name of the init container image
   * - initContainer.tag
     - The tag of the init container image 
   * - initContainer.pullPolicy
     - The policy to apply when pulling an image for init container deployment
   * - shmemDir
     - This is the path to the Kubernetes local store where the shared memory
       database will be written
   * - useHostIpc
     - This sets the use of the host inter-process communication system.
       Defaults to true
   * - useHostPid
     - This sets the use of the host process ID system. Defaults to true
   * - resources
     - This allows the specifications of resources (CPU, memory) requires to run the
       container
   * - nodeSelector
     - This allows the specification of using specific nodes to run the pod
   * - affinity
     - This specifies the scheduling constraints of the pod
   * - tolerations
     - This specifies the tolerations of the pod for any system taints

.. [#] A given producer is given a name key that is used to identify that producer (e.g. auxtel).
.. [#] The characters >- are used after the key so that the CSCs can be specified in a list

.. NOTE:: The brokerIp, brokerPort and registryAddr of the env section are not
          overrideable in the producers.name.env section.
          The nexus3 of the image section is not overrideable in the producers.name.image section.
          Control of those items is on a site basis.
          All producers at a given site will always use the same information.

CSC Chart
---------

Instead of having charts for every CSC, we employ an approach of having one chart that handles all of the variations.
The chart supports image pull secrets, volume mounts (ones requiring a storage allocation and ones pointing to a NFS server), overriding the standard container entrypoint, secret injection and a number of configurations specific to the OpenSplice/DDS system.

The chart consists of the Job Kubernetes Workflows API, ConfigMap and PersistentVolumeClaim Kubernetes Config and Storage APIs.
The Job API is used to provide correct behavior when a CSC is sent to OFFLINE mode, the pod should not restart.
If the CSC dies for an unknown reason, not one caught by a FAULT state transition, a new pod will be started and the CSC will then come up in its lowest control state.
The old pod will remain in a failed state, but available for interrogation about the problem.
The ConfigMap API is used for specifying the override of the container entrypoint.
The PersistentVolumeClaim API is used for supporting volume mounts that require a storage allocation request.
Next, we will turn to the chart configuration descriptions.

.. list-table:: CSC Chart YAML Configuration
   :widths: 15 25
   :header-rows: 1

   * - YAML Key
     - Description
   * - image
     - This section holds the configuration of the CSC container image
   * - image.repository
     - The Docker registry name of the container image to use for the CSC
   * - image.tag
     - The tag of the container image to use for the CSC
   * - image.pullPolicy
     - The policy to apply when pulling an image for deployment
   * - image.nexus3
     - The tag name for the Nexus3 Docker repository secrets if private images
       need to be pulled
   * - namespace
     - This is the namespace in which the CSC will be placed
   * - useMulticast
     - This sets the use of the annotation the `multus` address binding needed for DDS
       communication
   * - env
     - This section holds a set of key, value pairs for environmental variables
   * - envSecrets
     - This section holds specifications for secret injection
   * - envSecrets.name
     - The label for the secret
   * - envSecrets.secretName
     - The name of the vault store reference. Uses the namespace attribute to construct
       the full name
   * - envSecrets.secretKey
     - The key in the vault store containing the necessary secret
   * - entrypoint
     - This key allows specification of a script to override the entrypoint
   * - pvcMountpoint
     - This section holds the information necessary to create a volume mount
       for the container.
   * - pvcMountpoint.name
     - A label identifier for the mountpoint
   * - pvcMountpoint.path
     - The path inside the container to mount
   * - pvcMountpoint.accessMode [#]_
     - This sets the required access mode for the volume mount.
   * - pvcMountpoint.ids
     - This section contains UID and GID overrides
   * - pvcMountpoint.ids.uid
     - An alternative UID for mounting
   * - pvcMountpoint.ids.gid
     - An alternative GID for mounting
   * - pvcMountpoint.claimSize
     - The requested physical disk space size for the volume mount
   * - nfsMountpoint
     - This section holds the information necessary to create a NFS mount for the container
   * - nfsMountpoint.name
     - A label identified for the mountpoint
   * - nfsMountpoint.containerPath
     - The path for the container mountpoint
   * - nfsMountpoint.readOnly
     - This sets if the NFS mount is read only or read/write
   * - nfsMountpoint.server
     - The hostname of the NFS server
   * - nfsMountpoint.serverPath
     - The path exported by the NFS server
   * - useExternalConfig
     - This sets whether to rely on the ConfigMap for OSPL configuration or the internal
       OSPL configuration
   * - osplVersion
     - This is the version of the OpenSplice library to run. It is used to set the
       location of the OSPL configuration file
   * - initContainer
     - This section sets the option use of an init container for DDS route fixing
   * - initContainer.repository
     - The Docker registry name of the init container image
   * - initContainer.tag
     - The tag of the init container image 
   * - initContainer.pullPolicy
     - The policy to apply when pulling an image for init container deployment
   * - shmemDir
     - This is the path to the Kubernetes local store where the shared memory
       database will be written
   * - useHostIpc
     - This sets the use of the host inter-process communication system.
       Defaults to true
   * - useHostPid
     - This sets the use of the host process ID system. Defaults to true
   * - resources
     - This allows the specifications of resources (CPU, memory) requires to run the
       container
   * - nodeSelector
     - This allows the specification of using specific nodes to run the pod
   * - affinity
     - This specifies the scheduling constraints of the pod
   * - tolerations
     - This specifies the tolerations of the pod for any system taints

.. [#] Definitions can be found `here <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes>`_.

.. NOTE:: The configurations that are associated with each chart do not represent the full range of component coverage.
          The `ArgoCD Configuration` handles that.

Packaging and Deploying Charts
------------------------------

The GitHub repository is configured to automatically package and publish charts to the `chart repository <https://lsst-ts.github.io/charts/>`_ using the GitHub `chart releaser <https://github.com/helm/chart-releaser>`_ action.
The publish happens only on branch merges to master.
