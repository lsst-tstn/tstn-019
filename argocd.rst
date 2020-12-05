ArgoCD Configuration
====================

The configuration and subsequent deployment of the control components is handled by the `ArgoCD <https://argoproj.github.io/argo-cd/>`_ system.
The code for the ArgoCD configuration is kept in the `ArgoCD Github repository <https://github.com/lsst-ts/argocd-csc>`_.
The deployment methodologies will be handled in forthcoming sections.
ArgoCD uses the concept of an app of apps (or chart of charts).
Each app requires a specific chart or charts to use in order to deploy. 

Each component has its own directory within the top-level ``apps`` directory.
This includes the cluster configuration, OSPL configuration, OSPL daemon, Kafka producers and the CSCs.
There are a few special apps which collect the main CSC component apps into a group (further called collector apps).
Those will be discussed later.
Some applications have extra support included that are not present in the application chart.
That extra support will be explained within the appropriate section.

The contents found within the application directories have roughly the same
content.

Chart.yaml
  This file specifies the name of the application with that key in the file.

requirements.yaml
  This file specifies the Helm chart to use including the version.

values.yaml
  This file contains base information that will apply to all site specific configuration.
  May not be present in all applications.

.. warning:: Any changes to the ``values.yaml`` will be seen by all sites at
             once, so give careful thought to adjustments there.

values-<site tag>.yaml
  This file contains site specific information for the configuration.
  It may override configuration provided in the ``values.yaml`` file.
  The supported sites listed in the following table and not all applications will have all sites supported.

.. list-table:: Supported Sites
   :widths: 10 20
   :header-rows: 1

   * - Site Tag
     - Site Location
   * - base-teststand
     - La Serena Base Data Center
   * - ncsa-teststand
     - NCSA Test Stand
   * - summit
     - Cerro Pachon Control System Infrastructure
   * - tucson-teststand
     - Tucson Test Stand

templates
  This directory may not appear in all configurations.
  It will contain other Kubernetes or ArgoCD APIs to support deployment.

All ``values*.yaml`` files start with the chart name the application uses as the
top-level key.
Further keys are specified in the same manner as the Helm chart configuration.
Examples will be provided below.

Cluster Configuration
---------------------

This application (``cluster-config``) contains a VaultSecret `Vault <https://www.vaultproject.io/>`_ API in the ``templates`` directory that handles setting up the access credential secrets for image pulling into each defined namespace.
This requires a configuration parameter that is outside the chart level configuration. 

.. list-table:: Cluster Configuration Extra YAML Configuration
   :widths: 10 20
   :header-rows: 1

   * - YAML Key
     - Description
   * - deployEnv
     - The site tag to use when setting up the namespace secrets

Collector Apps
--------------

Within the ArgoCD Github repository, there are currently four collector applications: ``auxtel``, ``eas`` ``maintel`` and ``obssys``.
The layout for these apps is different and explained here.

Chart.yaml
  This file contains the specification of a new chart that will deploy a group of CSCs.

values.yaml
  This file contains configuration parameters to fill out the application deployment.
  The keys will be discussed below.

templates/<collector app name>.yaml
  This file contains the ArgoCD Application API used to deploy the associated CSCs specified by the collector app configuration.
  One application is generated for each CSC listed in the configuration.

.. list-table:: Collector Application YAML Configuration
   :widths: 10 20
   :header-rows: 1

   * - YAML Key
     - Description
   * - spec
     - This section defines elements for cluster setup and ArgoCD location
   * - spec.destination
     - This section defines information for the deployment destination
   * - spec.destination.server
     - The name of the Kubernetes resource to deploy on
   * - spec.source
     - This section defines the ArgoCD setup to use
   * - spec.source.repoURL
     - The repository housing the ArgoCD configuration
   * - spec.source.targetRevision
     - The branch on the ArgoCD repository to use for the configuration
   * - env
     - This key sets the site tag for the deployment. Must be in quotes.
   * - cscs
     - This key holds a list of CSCs that are associated with the app
   * - noSim
     - This key holds a list of CSCs that do not have a simulator capability
   * - runAsSim
     - This key holds a list of CSCs that are run in simulator mode at the summit
   * - indexed
     - This key holds a set of key-value pairs for indexed components that specify the length of the component name
   * - indexed.<csc name>
     - This contains the numeric value specifying the length of the <csc name> key

Apps with Special Support
-------------------------

This section will detail any applications that require special support that is outside the supplied chart.

Hexapodsim
~~~~~~~~~~

The ``athexapod`` application requires the use of a simulated hexapod low-level controller when running in simulation mode.
This simulator (``hexapodsim``) is accessed by a specific IP address and port.
The ``hexapodsim`` app uses a Service from the Kubernetes Service APIs to setup the port.
Kubernetes conjoins that with the deployed pod IP in an environment variable: ``HEXAPOD_SIM_SERVICE_HOST``.
The ATHexapod CSC code uses that variable to set the proper connection information.
The ``hexapodsim`` application has its own chart that uses Deployment from the Kubernetes Workloads API.
This chart contains no OSPL features since the simulator does not require it.
The use of the Deployment allows the application to be brought up with the ``auxtel`` collector app, but remain up if the CSCs are taken down since those are run as Jobs.

Header Service
~~~~~~~~~~~~~~

Both the ``aheaderservice`` and ``ccheaderservice`` apps require the use of an Ingress and Service Kubernetes APIs.
This is only necessary while the header services apps leverage an internal web service for header file exposition.
Once the header service moves to using the S3 LFA, the extra APIs will be removed.

Examples
--------

ArgoCD level configuration files follow this general format.

::

  chart-name:
    chart-key1: values
    chart-key2: values
    ...

If a given application uses extra APIs for deployment, those configurations will
look like the following.

::

  api-key1: values
  api-key2: values
  ...

Refer to the appropriate `Helm Chart` section for chart level key descriptions.
API key descriptions are in this section.

Cluster Configuration
~~~~~~~~~~~~~~~~~~~~~

The main ``values.yaml`` file looks like:

::
  
  cluster-config:
    namespaces:
      - auxtel
      - eas
      - maintel
      - obssys
      - kafka-producers
      - ospl-daemon

This sets the namespaces for all sites.
This configuration can be overridden on a per site basis, but it is not recommended for summit environment.

The site specific configuration files only needs to contain the `deployEnv` keyword.
The ``values-ncsa-teststand.yaml`` is shown as an example.

::

  deployEnv: ncsa-teststand

If you want to override the list of namespaces for a particular site, this is how it would be done for a site specific file.

::

  cluster-config:
    namespaces:
      - test1
      - myspace
      - home

  deployEnv: tucson-teststand

OSPL Configuration
~~~~~~~~~~~~~~~~~~

This is the ``ospl-config`` directory within the ArgoCD repository.
The all-site configuration in ``values.yaml`` looks like this.

::

  ospl-config:
    namespaces:
      - auxtel
      - eas
      - maintel
      - obssys
      - kafka-producers
      - ospl-daemon
    networkInterface: net1
    shmemSize: 504857600
    maxSamplesWarnAt: 50000
    schedulingClass: Default
    schedulingPriority: 0
    monitorStackSize: 6000000
    waterMarksWhcHigh: 8MB
    deliveryQueueMaxSamples: 10000
    squashParticipants: true
    namespacePolicyAlignee: Lazy
    domainReportEnabled: false
    ddsi2TracingEnabled: false
    ddsi2TracingVerbosity: finer
    ddsi2TracingLogfile: stdout
    durabilityServiceTracingEnabled: false
    durabilityServiceTracingVerbosity: FINER
    durabilityServiceTracingLogfile: stdout
 
The list of namespaces *MUST* contain at least the same namespaces as
``cluster-config``.
The `networkInterface` is the name specified by the ``multus`` CNI and is the same for all sites that we currently deploy to.
The rest of the configuration is meant for handling setup, services and features
related to the shared memory configuration.
If you want to adjust configuration parameters for testing without effecting
other sites, a site specific configuration file can be used.

OSPL Daemon Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~

The OSPL daemon configuration has a global ``values.yaml`` file that sets the
namespace, OSPL log files and OSPL version for all sites.
All other configuration should be handled in a site YAML configuration file.
The configuration from the ``values-summit.yaml`` configuration file is shown below.

::

  ospl-daemon:
    image:
      repository: ts-dockerhub.lsst.org/ospl-daemon
      tag: c0016
      pullPolicy: Always
      nexus3: nexus3-docker
    env:
      LSST_DDS_PARTITION_PREFIX: summit
    shmemDir: /run/ospl

Kafka Producer Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Kafka producer configuration has a global ``values.yaml`` file that sets the
namespace, OSPL log files and OSPL version and producer CSC configuration for all sites.
A snippet of the configuration is shown below.

::

  kafka-producers:
    namespace: kafka-producers
    ...
    producers:
      auxtel:
        cscs: >-
          ATAOS
          ATDome
          ATDomeTrajectory
          ATHexapod
          ATPneumatics
          ATPtg
          ATMCS
      maintel:
        cscs: >-
          MTAOS
          Dome
          MTDomeTrajectory
          MTPtg
      ...

Each key under `producers` is the name for that given producer along with the
list of CSCs that producer will monitor. 

The Docker image and other producer configuration is handled on a site basis.
Here is an example from the ``values-ncsa-teststand.yaml``. 

::

  kafka-producers:
    image:
      repository: ts-dockerhub.lsst.org/salkafka
      tag: c0016
      pullPolicy: Always
      nexus3: nexus3-docker
    env:
      lsstDdsDomain: ncsa
      brokerIp: cp-helm-charts-cp-kafka-headless.cp-helm-charts
      brokerPort: 9092
      registryAddr: https://lsst-schema-registry-nts-efd.ncsa.illinois.edu
      partitions: 1
      replication: 3
      waitAck: 1
      logLevel: 20

The `env` information is specifically tailored for the NCSA teststand.
The  `image` information is applied to all producers at this site.
You can override both the producers deployed, reconfigure them if necessary or add new ones to a specific site.
You can also change the image information for a given producer as well.
You must ensure that the different image can interact with the others already deployed without interfering with their functioning.
Below is an example of doing all the above.

::

  kafka-producers:
    image:
      repository: ts-dockerhub.lsst.org/salkafka
      tag: c0016
      pullPolicy: Always
      nexus3: nexus3-docker
    env:
      lsstDdsDomain: ncsa
      brokerIp: cp-helm-charts-cp-kafka-headless.cp-helm-charts
      brokerPort: 9092
      registryAddr: https://lsst-schema-registry-nts-efd.ncsa.illinois.edu
      partitions: 1
      replication: 3
      waitAck: 1
      logLevel: 20
    producers:
      comcam: null
      auxtel: null
      eas:
        cscs: >-
          DSM
      latiss: null
      test: 
        image:
          tag: c0016.001
      ccarchiver:
        cscs: >-
          CCArchiver
      cccamera:
        cscs: >-
          CCCamera
      ccheaderservice:
        cscs: >-
          CCHeaderService

The `null` is how to remove producers from the ``values.yaml`` configuration. 
The ``eas`` producer changes the list of CSCs from DIMM, DSM, Environment to
DSM.
The ``test`` producer changes the site configured image tag to something different.
The ``ccarchiver``, ``cccamera`` and ``ccheaderservice`` producers are new ones specified for this site only.

CSC Configuration
~~~~~~~~~~~~~~~~~

There are few different variants of CSC configuration as discussed previously. 
Most CSC configuration consists of Docker image information and environment variables that must be set as well as the namespace that the CSC should belong to.
The namespace is handled in the CSC ``values.yaml`` in order to have that applied uniformly across all sites.
An example of a simple configuration showing a specific namespace is shown below.

::

  csc:
    namespace: maintel

CSCs may have other values they need to apply regardless of site. Here is an
example from the ``mtcamhexapod`` application.

::

  csc:
    namespace: maintel
    env:
      OSPL_INFOFILE: /tmp/ospl-info-mtcamhexapod.log
      OSPL_ERRORFILE: /tmp/ospl-error-mtcamhexapod.log
    osplVersion: V6.10.4

Other global environment variables can be specified in this manner.

The Docker image configuration is handled on a site basis to allow independent evolution.
This also applies to the ``LSST_DDS_PARTITION_PREFIX`` environment variable since those are definitely site specific.
Below is an example site configuration from the ``mtcamhexapod`` for the NCSA test stand.

::

  csc:
    image:
      repository: ts-dockerhub.lsst.org/mthexapod
      tag: c0016
      pullPolicy: Always
      nexus3: nexus3-docker
    env:
      LSST_DDS_PARTITION_PREFIX: ncsa
      RUN_ARG: --simulate 1
    shmemDir: /scratch.local/ospl

The ``RUN_ARG`` configuration sets the index for the underlying component that the container will run and puts it into simulation mode.
Other site specific environment variables can be listed in the `env` section if they are appropriate to running the CSC container.

All containers require the use of the Nexus3 repository, identified by the use of ``ts-dockerhub.lsst.org`` in the `image.repository` name.
The `image.nexus3` key must be configured in order for secret access to occur.
The value in the `image.nexus3` entry is specific to the Nexus3 instance that is based in Tucson.
This may be expanded to other replications in the future.
The site specific configuration for the ``mtcamhexapod`` application given previously shows this information is configured.

.. warning:: The entrypoint configuration is currently broken and needs to be 
             fixed in the current CSC Helm chart.
             There has been less use of this feature recently, so this may be retired.
             The documentation below will not be removed for now.

The CSC container may need to override the command script that the container automatically runs on startup.
An example of how this is accomplished is shown below.

::

  csc: 
    image:
      repository: ts-dockerhub.lsst.org/atdometrajectory
      tag: c0016
      pullPolicy: Always

    env:
      LSST_DDS_PARTITION_PREFIX: lsatmcs

    entrypoint: |
      #!/usr/bin/env bash

      source ~/miniconda3/bin/activate

      source $OSPL_HOME/release.com
      
      source /home/saluser/.bashrc

      run_atdometrajectory.py

The script for the `entrypoint` must be entered line by line with an empty line between each one in order for the script to be created with the correct execution formatting.
The pipe (|) at the end of the `entrypoint` keyword is required to help obtain the proper formatting.
Using the `entrypoint` key activates the use of the ConfigMap API.

.. note:: End currently broken feature documentation.

If a CSC requires a physical volume to write files out to, the `mountpoint` key should be used.
This should be a rarely used variant, but it is supported.
The persistent volume claim is local to the Kubernetes cluster and by default is not persisted if the volume claim disappears. 
The Header Service will use this when deployed to the summit until the S3 system is available.
A configuration might look like the following.

::

  csc:
    ...
    mountpoint:
      - name: www
        path: /home/saluser/www
        accessMode: ReadWriteOnce
        claimSize: 50Gi

The description of the `claimSize` units can be found at this `page <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory>`_.

Collector Applications
~~~~~~~~~~~~~~~~~~~~~~

As noted earlier, these applications are collections of individual CSC apps aligned with a particular subsystem.
The all-site configuration handles ArgoCD information, CSCs that are not simulators and possibly indexed CSCs.
Here is how the ``values.yaml`` file for the ``maintel`` app looks.

::

  spec:
    destination:
      server: https://kubernetes.default.svc
    source:
      repoURL: https://github.com/lsst-ts/argocd-csc
      targetRevision: HEAD

  noSim:
    - mtptg
    - mtdometrajectory

The `spec` section is specific to ArgoCD and should not be changed unless you really understand the consequences. The exceptions to this are the `repoURL` and `targetRevision` parameters.
It is possible the Github repository moves during the lifetime of the project, so `repoURL` will need to be updated if that happens.
There might also be a need to testing something that is not on the ``master`` branch of
the repository.
To support that, change the `targetRevision` to the appropriate branch name.
Use this sparingly, as main configuration tracking is on the ``master`` branch.

The site specific configurations handle setting up the list of CSCs to run and the environment for that site.
It can also handle the need for running simulators on the summit.
Below is the configuration for ``maintel`` from the ``values-summit.yaml`` configuration.

::

  env: summit
  cscs:
    - mtaos
    - mtptg
    - mtmount
    - mtdome
    - mtdometrajectory
    - ccheaderservice
  runAsSim:
    - mtaos
    - mtdome
    - mtmount

The `env` parameter sets the ``value-<env>.yaml`` for the listed CSC apps.
This will change on a per site basis. The `cscs` parameter is the listing of the CSC apps that the collector app will control.

The ``auxtel`` collector app follows similar configuration mechanisms but controls a different list of CSC apps as does the ``obssys`` and ``eas`` collector apps.

The ``eas`` collector app has one other variation since it handles indexed CSCs all coming from the same application.
Below is a section from the ``values.yaml`` file showing how indexed components are treated.

::

  ...
  indexed:
    dimm: 4
    weatherstation: 14
    dsm: 3

The key points to the application directory that will be used and the number represents the length of the key.
This is necessary since there are no tools in the Helm system that allow the length determination of a string.
To use the indexed components, here is the ``values-summit.yaml`` configuration for the ``eas`` app.

::

  env: summit
  cscs:
    - dimm1
    - dimm2

Using the length provided in the ``values.yaml`` file, the number at the end of the name is retrieved.
This is used to set the appropriate configuration file in the application directory for the specific index.
The index configuration files look like ``values-<app name><#>.yaml`` and contain index specific setup.
The index=1 DIMM component configuration ``values-dimm1.yaml`` is shown below.

::

  csc:
    env:
      RUN_ARG: 1

The index specific configuration required will vary with the different CSCs.
