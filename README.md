# OpenShift Hosted Control Planes w/ Multi-Arch

A hosted control plane (HCP) is a cloud-native architecture where the management components of a Red Hat® OpenShift® cluster, specifically the control plane, are decoupled from the worker nodes and managed as a service. HCP offers a consolidated, efficient, and secure approach to managing OpenShift and other Kubernetes clusters at scale. Instead of running on dedicated infrastructure (for the masters) within each cluster, the control plane components are hosted on a separate management cluster and managed as regular OpenShift workloads. This separation offers many advantages for organizations looking to optimize their OpenShift deployments especially for cost, strong isolation, and fast cluster provisioning time.

Some of the benefits of hosted control planes are as follows:

* Reduced Costs:
* Fast Provisioning:
* Isolation:
* Scalability

All of these benefits make HCP an attractive solution for businesses looking to get the most value out of their infrastructure.   The rest of this blog will cover the process of configuring and then deploy an HCP cluster.  First let's take a look at the environment we are working with so we understand what we are starting from.

## Environment

The base environment started with an x86 architecture of OpenShift 4.20.8 installed in a hyperconverged three node control/worker setup on virtual machines.  The environment is depicted in the followng diagram:




These three nodes also have Red Hat OpenShift Data Foundation installed to provide the backing storage for MultiCluster Engine which is the basis for HCP.  We have also gone ahead and installed but not configured the following operators: MultiCluster Engine and MetalLB.  Since we are going to be deploying an HCP cluster made up of Arm worker nodes let's first confirm a few things in the environment.

First let's confirm that the MultiCluster Engine and MetalLB operators are installed.

~~~bash
$ oc get csv -n multicluster-engine
NAME                                    DISPLAY                              VERSION               REPLACES   PHASE
metallb-operator.v4.20.0-202512222252   MetalLB Operator                     4.20.0-202512222252              Succeeded
multicluster-engine.v2.10.0             multicluster engine for Kubernetes   2.10.0                           Succeeded
~~~

Next let's make sure our hyperconverged cluster has multi architecture support.

~~~bash
$ oc adm release info -o jsonpath="{ .metadata.metadata}"
{"url":"https://access.redhat.com/errata/RHBA-2025:23103"}
~~~

From the output above it appears we are not set up for multi architecture.  But that is an easy fix because we can enable that as a day two operator.  Running the following command should resolve our issue.

~~~bash
$ oc adm upgrade --to-multi-arch
Requested update to multi cluster architecture
~~~

After a few minutes we can run our multi architecture command to check again.

~~~bash
$ oc adm release info -o jsonpath="{ .metadata.metadata}"
{"release.openshift.io/architecture":"multi","url":"https://access.redhat.com/errata/RHBA-2025:23103"
~~~

Now our cluster looks good for us to move forward in our journey.

## Configuring MultiCluster Engine




