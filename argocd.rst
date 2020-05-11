ArgoCD Configuration
====================

The configuration and subsequent deployment of the control components is
handled by the `ArgoCD <https://argoproj.github.io/argo-cd/>`_ system. The
code for the ArgoCD configuration is kept in the
`ArgoCD Github repository <https://github.com/lsst-ts/argocd-csc>`_. The
deployment methodologies will be handled in forthcoming sections. ArgoCD uses
the concept of an app of app (or chart of charts). Each app requires a specfic
chart or charts to use in order to deploy. 

Each component has its own directory within the top-level `apps` directory. This
includes the Kafka producers. There are a few special apps (further called
collector apps) which collect the main CSC component apps into a group. Those
will be discussed later. 

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
  override configuration provided in the `values.yaml` file. The supported sites
  listed in the following table and not all applications will have all sites
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
   * - sandbox
     - Currently a GKE instance, soon to be replaced by LSP-int at NCSA
   * - summit
     - Cerro Pachon Control System Infrastructure
   * - tucson-teststand
     - Tucson Test Stand. This is now largely defunct

templates
  This directory contains a `<app-name>-ns.yaml` file defining a Kubernetes
  Cluster API: Namespace. This defines a specific namespace for the application.

All `values*.yaml` files start with the chart name the application uses as the
top-level key. Further keys are specified in the same manner as the Helm chart
configuration.

Collector Apps
--------------

Within the ArgoCD Github repository, there are currently two collector
applications: `auxtel` and `maintel`. The layout for these apps is different and
explained here.

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

