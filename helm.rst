Helm Charts
===========

The packaging of the CSC deployment starts with a `Helm <https://v2.helm.sh/>`_
chart. We using version 2 of Helm due the current ArgoCD restriction on the Helm
version. The code for the charts are kept in the
`Helm chart Github repository <https://github.com/lsst-ts/charts>`_. The next
two sections will discuss each chart in detail. For a description of the APIs
used, consult the
`Kubernetes documentation <https://kubernetes.io/docs/reference/>`_ for the API
reference. The chart sections will not go into great detail on the content of
each API delivered. Each chart section will list all of the possible
configuration aspects that each chart is delivering, but full use of that
configuration is left to the `ArgoCD Configuration` section. For the CSC
deployment, we will run a single container per pod on Kubernetes. The Kafka
producers may follow a similar pattern, but they are allowed to scale beyond
the single pod?

Kafka Producer Chart
--------------------

While not a true control component, the Kafka producers are nevertheless an
important part of the control system landscape. They have the capability to
convert the SAL messages into Kafka messages that are then ingested into the
Engineering Facilities Database (EFD). See :cite:`SQR-034` for more details. 

The chart consists of a single Kubernetes Workloads API: Deployment. The
Deployment API allows for restarts if a particular pod dies which assists in
keeping the producers up and running all the time. For each producer specified
in the configuration, a deployment will be created. We will now cover the
configuration options for the chart.

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
   * - env
     - This section holds environment configuration for the producer container
   * - env.lsstDdsDomain
     - The LSST_DDS_DOMAIN name applied to all producer containers
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
   * - env.logLevel
     - This value determines the logging level for the producers
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
   * - producers.name.env.lsstDdsDomain
     - The LSST_DDS_DOMAIN name applied the named producer container
   * - producers.name.env.partitions
     - The number of partitions that the named producer is supporting
   * - producers.name.env.replication
     - The number of replications available to the named producer
   * - producers.name.env.logLevel
     - This value determines the logging level for the named producer     

.. [#] A given producer is given a name key that is used to identify that producer (e.g. auxtel).
.. [#] The characters >- are used after the key so that the CSCs can be specified in a list

.. NOTE:: The brokerIp, brokerPort and registryAddr of the env section are not
          overrideable in the producers.name.env section. Control of those items
          is on a site basis. All producers at a given site will always use the
          same information.

CSC Chart
---------

Instead of having charts for every CSC, we employ an approach of having one
chart that describes all the different CSC variants. There are four main
variants that the chart supports:

simple
  A CSC that requires no special interventions and uses only environment
  variables for configuration

entrypoint
  A CSC that uses an override script for the container entrypoint.

imagePullSecrets
  A CSC that requires the use of the Nexus3 repository and need access
  credential for pulling the associated image

volumeMount
  A CSC that requires access to a physical disk store in order to transfer
  information into the running container

The chart consists of the Job Kubernetes Workflows API, ConfigMap and
PersistentVolumeClaim Kubernetes Config and Storage APIs and VaultSecret
`Vault <https://www.vaultproject.io/>`_ API. The Job API is used to provide
correct behavior when a CSC is sent of OFFLINE mode, the pod should not restart.
The drawback to this is if a CSC dies for an unknown reason, not one caught by
FAULT state transition, the pod will not restart and requires startup
intervention. The other three APIs are used to support the non-simple CSC
variants. They will be mentioned in the configuration description which we will
turn to next.

.. warning:: The volumeMount variant is still in the development phase, so it is
             currently not supported.

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
   * - image.useNexus3 [#]_
     - This key activates the VaultSecret API for secure image pulls
   * - env [#]_
     - This section holds a set of key, value pairs for environmental variables
   * - entrypoint [#]_
     - This key allows specification of a script to override the entrypoint
   * - deployEnv [#]_
     - This key assists the VaultSecret in knowing where to look for credentials

.. [#] The value of the key is not used, but use true for the value
.. [#] See env example below
.. [#] Format is important. See entrypoint example below
.. [#] The name is site specific and handled in the ArgoCD configuration

Example env YAML section

::

  env:
    LSST_DDS_DOMAIN: mytest
    CSC_INDEX: 1
    CSC_MODE: 1

The section can contain any number of environmental variables that are
necessary for CSC configuration.

Example entrypoint YAML section

::

  entrypoint: |
  #!/usr/bin/env bash

  source ~/miniconda3/bin/activate

  source $OSPL_HOME/release.com

  source /home/saluser/.bashrc

  run_atdometrajectory.py

The script must be entered line by line with an empty line between each one in
order for the script to be created with the correct execution formatting. The
pipe (|) at the end of the entrypoint keyword is required to help obtain the
proper formatting. Using the entrypoint key activates the use of the ConfigMap
API.

.. NOTE:: The configurations that are associated with each chart do not
          represent the full range of component coverage. The
          `ArgoCD Configuration`
          handles that.

Packaging and Deploying Charts
------------------------------

The Github repository has a README that contains information in how to package
up a new chart for deployment to the
`chart repository <https://lsst-ts.github.io/charts/>`_. First, ensure that the
chart version has been updated in the `Chart.yaml` file. The step for
creating/updating the index file needs one more flag for completeness.

::

  helm repo index --url=https://lsst-ts.github.io/charts .

Once the version number is updated, the chart packaged and the index file
updated, they can be collected into a single commit and pushed to master. That
push to master will trigger the installation of the new chart into the chart
repository. 
