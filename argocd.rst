ArgoCD Configuration
====================

The configuration and subsequent deployment of the control components is
handled by the `ArgoCD <https://argoproj.github.io/argo-cd/>`_ system. The
code for the ArgoCD configuration is kept in the
`ArgoCD Github repository <https://github.com/lsst-ts/argocd-csc>`_. The
deployment methodologies will be handled in forthcoming sections. ArgoCD uses
the concept of an app of apps (or chart of charts). Each app requires a specific
chart or charts to use in order to deploy. 

Each component has its own directory within the top-level ``apps`` directory.
This includes the cluster configuration, OSPL configuration, Kafka producers and
the CSCs. There are a few special apps, cluster configuration and app which
collect the main CSC component apps into a group (further called collector
apps). Those will be discussed later. 

The contents found within the application directories have roughly the same
content.

Chart.yaml
  This file specifies the name of the application with that key in the file.

requirements.yaml
  This file specifies the Helm chart to use including the version.

values.yaml
  This file contains base information that will apply to all site specific
  configuration. May not be present in all applications.

values-<site tag>.yaml
  This file contains site specific information for the configuration. It may
  override configuration provided in the ``values.yaml`` file. The supported
  sites listed in the following table and not all applications will have all sites
  supported.

.. list-table:: Supported Sites
   :widths: 10 20
   :header-rows: 1

   * - Site Tag
     - Site Location
   * - base
     - La Serena Base Data Center
   * - ncsa-teststand
     - NCSA Test Stand
   * - ncsa-lsp-int
     - LSP-int at NCSA
   * - summit
     - Cerro Pachon Control System Infrastructure
   * - tucson-teststand
     - Tucson Test Stand. This is now largely defunct

templates
  This directory may not appear in all configurations. It will contain other
  Kubernetes or ArgoCD APIs to support deployment.

All ``values*.yaml`` files start with the chart name the application uses as the
top-level key. Further keys are specified in the same manner as the Helm chart
configuration. Examples will be provided below.

Cluster Configuration
---------------------

This application (``cluster-config``) contains a VaultSecret
`Vault <https://www.vaultproject.io/>`_ API in the ``templates`` directory that
handles setting up the access credential secrets for image pulling into each
defined namespace. This requires a configuration parameter that is outside the
chart level configuration. 

.. list-table:: Cluster Configuration Extra YAML Configuration
   :widths: 10 20
   :header-rows: 1

   * - YAML Key
     - Description
   * - deployEnv
     - The site tag to use when setting up the namespace secrets

The default configuration contains the following four namespaces and are used in
the Kafka producer and CSC applications.

- auxtel
- maintel
- obssys
- kafka-producers

Collector Apps
--------------

Within the ArgoCD Github repository, there are currently two collector
applications: ``auxtel`` and ``maintel``. The layout for these apps is different
and explained here.

Chart.yaml
  This file contains the specification of a new chart that will deploy a group
  of CSCs.

values.yaml
  This file contains configuration parameters to fill out the application
  deployment. The keys will be discussed below.

templates/<collector app name>.yaml
  This file contains the ArgoCD Application API used to deploy the associated
  CSCs specified by the collector app configuration. One application is
  generated for each CSC listed in the configuration.

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
      - maintel
      - obssys
      - kafka-producers

This sets the namespaces for all sites. This configuration can be overridden on
a per site basis, but it is not recommended for production environments such as
the summit, base and NCSA test stand.

The site specific configuration files only need to contain the `deployEnv`
keyword. The ``values-ncsa-teststand.yaml`` is shown as an example.

::

  deployEnv: ncsa-teststand

If one does want to override the list of namespaces for a particular site, this
is how it would be done for a site specific file.

::

  cluster-config:
    namespaces:
      - test1
      - myspace
      - home

  deployEnv: tucson-teststand

OSPL Configuration
~~~~~~~~~~~~~~~~~~

This is the ``ospl-config`` directory within the ArgoCD repository. There is one
and only one configuration for this application.

::

  ospl-config:
    namespaces:
      - auxtel
      - maintel
      - obssys
      - kafka-producers

    networkInterface: net1

The list of namespaces MUST contain at least the same namespaces as
``cluster-config``. The `networkInterface` is the name specified by the
``multus`` CNI and is the same for all sites that we currently deploy to.

Kafka Producer Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Kafka producer configuration has a global ``values.yaml`` file that sets the
namespace and producer CSC configuration for all sites. A snippet of the 
configuration is shown below.

::

  kafka-producers:
    namespace: kafka-producers

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

.. warning:: Any changes to the ``values.yaml`` will be seen by all sites at
             once, so give careful thought to adjustments there.

The Docker image and other producer configuration is handled on a site basis.
Here is an example from the ``values-ncsa-teststand.yaml``. 

::

  kafka-producers:
    image:
      repository: lsstts/salkafka
      tag: v1.1.2_salobj_v5.11.0_xml_v5.1.0
      pullPolicy: Always

    env:
      lsstDdsDomain: ncsa
      brokerIp: cp-helm-charts-cp-kafka-headless.cp-helm-charts
      brokerPort: 9092
      registryAddr: https://lsst-schema-registry-nts-efd.ncsa.illinois.edu
      partitions: 1
      replication: 3
      logLevel: 20

The `env` information is specifically tailored for the NCSA teststand. The 
`image` information is applied to all producers at this site. You can override
both the producers deployed, reconfigure them if necessary or add new ones to
a specific site. You can also change the image information for a given producer
as well. You must ensure that the different image can interact with the others
already deployed without interfering with their functioning. Below is an example
of doing all the above.

::

  kafka-producers:
    image:
      repository: lsstts/salkafka
      tag: v1.1.2_salobj_v5.11.0_xml_v5.1.0
      pullPolicy: Always

    env:
      lsstDdsDomain: ncsa
      brokerIp: cp-helm-charts-cp-kafka-headless.cp-helm-charts
      brokerPort: 9092
      registryAddr: https://lsst-schema-registry-nts-efd.ncsa.illinois.edu
      partitions: 1
      replication: 3
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
          tag: v1.1.3_salobj_v5.12.0_xml_v5.2.0
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
DSM. The ``test`` producer changes the site configured image tag to something
different. The ``ccarchiver``, ``cccamera`` and ``ccheaderservice`` producers
are new ones specified for this site only.

CSC Configuration
~~~~~~~~~~~~~~~~~

There are few different variants of CSC configuration as discussed previously. 
Most CSC configuration consists of Docker image information and environment 
variables that must be set as well as the namespace that the CSC should belong
to. The namespace is handled in the CSC ``values.yaml`` in order to have that
applied uniformly across all sites. An example of a simple one showing a 
specific namespace is shown below.

::

  csc:
    namespace: maintel

CSCs may have other values they need to applied regardless of site. Here is an
example from the ``mtcamhexapod`` application.

::

  csc:
    env:
      RUN_ARG: -s 1

    namespace: maintel

The ``RUN_ARG`` configuration sets the index for the underlying component that
the container will run. Other global environment variables can be specified in
this manner.

The Docker image configuration is handled on a site basis to allow independent
evolution. This also applies to the ``LSST_DDS_DOMAIN`` environment variable
since those are definitely site specific. Below is an example site configuration
from the ``mtcamhexapod`` for the NCSA test stand.

::

  csc: 
    image:
      repository: lsstts/hexapod
      tag: v0.5.2
      pullPolicy: Always

    env:
      LSST_DDS_DOMAIN: ncsa 

Other site specific environment variables can be listed in the `env` section if
they are appropriate to running the CSC container.

Containers that require the use of the Nexus3 repository, currently identified
by the use of ``ts-dockerhub.lsst.org`` in the `image.repository` name, need to
configure the `image.nexus3` key in order for secret access to occur. An example 
``values.yaml`` file for the ``mtptg`` is shown below.

::

  csc:
    image:
      nexus3: nexus3-docker

    env:
      TELESCOPE: MT

    namespace: maintel

The value in the `image.nexus3` entry is specific to the Nexus3 instance that
is based in Tucson. This may be expanded to other replications in the future.

The CSC container may need to override the command script that the container
automatically runs on startup. An example of how this is accomplished is shown
below.

::

  csc: 
    image:
      repository: lsstts/atdometrajectory
      tag: v1.2_salobj_v5.4.0_idl_v1.1.2_xml_v4.7.0
      pullPolicy: Always

    env:
      LSST_DDS_DOMAIN: lsatmcs

    entrypoint: |
      #!/usr/bin/env bash

      source ~/miniconda3/bin/activate

      source $OSPL_HOME/release.com
      
      source /home/saluser/.bashrc

      run_atdometrajectory.py

The script for the `entrypoint` must be entered line by line with an empty line
between each one in order for the script to be created with the correct
execution formatting. The pipe (|) at the end of the `entrypoint` keyword is
required to help obtain the proper formatting. Using the `entrypoint` key
activates the use of the ConfigMap API.

If a CSC requires a physical volume to write files out to, the `mountpoint` key
should be used. This should be a rarely used variant, but it is supported. The
Header Service will use this when deployed to the summit until the S3 system
is available. A configuration might look like the following.

::

  csc:
    ...

    mountpoint:
      - name: www
        path: /home/saluser/www
        accessMode: ReadWriteOnce
        claimSize: 50Gi

The description of the `claimSize` units can be found at this
`page <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory>`_.

Collector Applications
~~~~~~~~~~~~~~~~~~~~~~

As noted earlier, these applications are collections of individual CSC apps
aligned with a particular subsystem. The main configuration is the list of CSC
apps to include on launch. Here is how the ``values.yaml`` file for the
``maintel`` app looks.

::

  spec:
    destination:
      server: https://kubernetes.default.svc
    source:
      repoURL: https://github.com/lsst-ts/argocd-csc
      targetRevision: HEAD

  env: ncsa-teststand

  cscs:
    - mtaos
    - mtcamhexapod
    - mtm1m3
    - mtm2
    - mtm2hexapod
    - mtmount
    - mtptg
    - mtrotator

  noSim:
    - mtptg

The `spec` section is specific to ArgoCD and should not be changed unless you
really understand the consequences. The exceptions to this are the `repoURL`
and `targetRevision` parameters. It is possible the Github repository moves
during the lifetime of
the project, so `repoURL` will need to be updated if that happens. There might
also be a need to testing something that is not on the ``master`` branch of
the repository. To support that, change the `targetRevision` to the
appropriate branch name. Use this sparingly, as main configuration tracking is
on the ``master`` branch. The `env` parameter sets the ``value-<env>.yaml`` for
the listed CSC apps. This will change on a per site basis. The `cscs` parameter
is the listing of the CSC apps that the collector app will control. This can
also be changed on a per site basis.

As an example of per site configuration, below is an example for the summit
configuration of the ``maintel`` app.

::

  env: summit

  cscs:
    - mtaos
    - mtptg

As you can see, the `env` parameter is overridden to the correct name and the
list of CSCs is much shorter. This is due to the presence of real hardware on
the summit. The ``auxtel`` collector app follows similar configuration
mechanisms but controls a different list of CSC apps.
