:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. note:: This document presents the infrastructure and operation of
          containerized component deployment for the Vera C. Rubin Observatory.


.. sectnum::

Introduction
============

Many of the control system components, but not all, have been developed for
deployment via Docker containers. Initially we leveraged `docker-compose` as a
mechanism for prototyping container deployment to host services. With the
proliferation of Kubernetes (k8s) clusters around the Vera C. Rubin Observatory
and the associated sites, a new style of deployment and configuration is needed.
We will leverage the experience SQuaRE has gained deploying the EFD and Nublado
services and use a similar system based on Helm charts and ArgoCD configuration
and deployment. The following sections will discuss the two key components and
then will be expanded into including operation aspects as experience on that
front is gained.

.. include:: helm.rst
.. include:: argocd.rst


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib
                  lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
