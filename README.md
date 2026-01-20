# OpenShift Hosted Control Planes w/ Multi-Arch

A hosted control plane (HCP) is a cloud-native architecture where the management components of a Red Hat® OpenShift® cluster, specifically the control plane, are decoupled from the worker nodes and managed as a service. HCP offers a consolidated, efficient, and secure approach to managing OpenShift and other Kubernetes clusters at scale. Instead of running on dedicated infrastructure (for the masters) within each cluster, the control plane components are hosted on a separate management cluster and managed as regular OpenShift workloads. This separation offers many advantages for organizations looking to optimize their OpenShift deployments especially for cost, strong isolation, and fast cluster provisioning time.

Some of the benefits of hosted control planes are as follows:

* Reduced Costs: Smaller resource footprint and efficient resource utilization increases ROI
* Fast Provisioning: Control plane containers spin up much faster than first having to deploy RHCOS on baremetal or virtual machines
* Isolation: Dedicated infrastructure and security for the control plane enhance isolation, minimize attack surfaces, and improve overall security posture.
* Scalability: The decoupled architecture enables independent scaling of control plane and worker nodes.

All of these benefits make HCP an attractive solution for businesses looking to get the most value out of their infrastructure.   The rest of this blog will cover the process of configuring and then deploy an HCP cluster.  First let's take a look at the environment we are working with so we understand what we are starting from.

## Environment

The base environment started with an x86 architecture of OpenShift 4.20.8 installed in a hyperconverged three node control/worker setup on virtual machines.  The environment is depicted in the followng diagram:




These three nodes also have Red Hat OpenShift Data Foundation installed to provide the backing storage for MultiCluster Engine which is the basis for HCP.  Since we are going to be deploying an HCP cluster made up of Arm worker nodes let's first confirm the cluster has multi architecture enabled.

We can confirm multi-archiecture is enabled on OpenShift by running the following command.

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

## Install & Configuring MultiCluster Engine Operator

One of the challenges of scaling OpenShift environments is managing the lifecycle of a growing fleet. To meet that challenge, we can use the Multicluster Engine Operator. The operator delivers full lifecycle capabilities for managed OpenShift Container Platform clusters and partial lifecycle management for other Kubernetes distributions. It is available in two ways:

* As a standalone operator that you install as part of your OpenShift Container Platform or OpenShift Kubernetes Engine subscription
* As part of Red Hat Advanced Cluster Management for Kubernetes 

For Hosted Control Planes this operator is required and for this demonstration we will us it in standalone mode.   The first step is to install the operator with the following custom resource file.

~~~bash
$ cat <<EOF >multicluster-engine-operator.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: multicluster-engine
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: multicluster-engine
  namespace: multicluster-engine
spec:
  targetNamespaces:
  - multicluster-engine
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: multicluster-engine
  namespace: multicluster-engine
spec:
  channel: stable-2.10
  installPlanApproval: Automatic
  name: multicluster-engine
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

One we have generated the custom resource file we can create it on the cluster.

~~~bash
$ oc create -f multicluster-engine-operator.yaml 
namespace/multicluster-engine created
operatorgroup.operators.coreos.com/multicluster-engine created
subscription.operators.coreos.com/multicluster-engine created
~~~

We can verify the dual instances of the operator is up and running by running the following command.

~~~bash
$ oc get pods -n multicluster-engine
NAME                                           READY   STATUS    RESTARTS   AGE
multicluster-engine-operator-6dd66fff8-gphcf   1/1     Running   0          9m20s
multicluster-engine-operator-6dd66fff8-tq6mx   1/1     Running   0          9m20s
~~~

Now that the operator is up and running we need to go ahead and create a multicluster engine instance.   The following custom resource file contains the values to create that instance.

~~~bash
$ cat <<EOF >multicluster-engine-instance.yaml
apiVersion: multicluster.openshift.io/v1
kind: MultiClusterEngine
metadata:
  name: multiclusterengine
spec:
  availabilityConfig: Basic
  targetNamespace: multicluster-engine
EOF
~~~

~~~bash
$ oc create -f multicluster-engine-instance.yaml 
multiclusterengine.multicluster.openshift.io/multiclusterengine created
~~~

~~~bash
$ oc get pods -n multicluster-engine
NAME                                                   READY   STATUS    RESTARTS   AGE
cluster-curator-controller-7c66f8b67f-hbhkr            1/1     Running   0          8m30s
cluster-image-set-controller-6879c9fdf7-vhvsp          1/1     Running   0          8m29s
cluster-manager-847d499df7-kb5bx                       1/1     Running   0          8m29s
cluster-manager-847d499df7-w2sdj                       1/1     Running   0          8m29s
cluster-manager-847d499df7-z65kp                       1/1     Running   0          8m29s
cluster-proxy-addon-manager-86484759b9-mhgpg           1/1     Running   0          6m38s
cluster-proxy-addon-user-5fff4bbf8-57r7v               2/2     Running   0          6m38s
cluster-proxy-fbf4447f4-ch8p9                          1/1     Running   0          5m
clusterclaims-controller-dfcf6dcd4-b4p44               2/2     Running   0          8m29s
clusterlifecycle-state-metrics-v2-7c66dbd6f9-pslqq     1/1     Running   0          8m30s
console-mce-console-7dbbc66784-bb292                   1/1     Running   0          8m32s
discovery-operator-7997f54695-6mdct                    1/1     Running   0          8m31s
hcp-cli-download-5c4dfbfd6c-lgdhz                      1/1     Running   0          4m59s
hive-operator-6545b5986b-6pttn                         1/1     Running   0          8m31s
hypershift-addon-manager-64797b9868-h26wg              1/1     Running   0          6m44s
infrastructure-operator-5f9d89c69-k9b82                1/1     Running   0          8m30s
managedcluster-import-controller-v2-75b55d65bd-4h8b4   1/1     Running   0          8m27s
multicluster-engine-operator-6dd66fff8-gphcf           1/1     Running   0          25m
multicluster-engine-operator-6dd66fff8-tq6mx           1/1     Running   0          25m
ocm-controller-84964b45bb-h5hvs                        1/1     Running   0          8m28s
ocm-proxyserver-8cbffb748-mj5hx                        1/1     Running   0          8m26s
ocm-webhook-7d99759b8d-5dv9j                           1/1     Running   0          8m28s
provider-credential-controller-6f54b788b5-zm9bd        2/2     Running   0          8m30s
~~~

~~~bash
$ oc patch mce multiclusterengine --type=merge -p   '{"spec":{"overrides":{"components":[{"name":"hypershift","enabled": true}]}}}'
multiclusterengine.multicluster.openshift.io/multiclusterengine patched
~~~

~~~bash
$ cat <<EOF >agent-service-config.yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
  name: agent
spec:
  databaseStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  filesystemStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  imageStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
  osImages:
    - openshiftVersion: "4.20"
      version: 9.6.20260107-0
      url: "https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.20/latest/rhcos-4.20.0-x86_64-live-iso.x86_64.iso"
      rootFSUrl: "https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.20/latest/rhcos-4.20.0-x86_64-live-rootfs.x86_64.img"
      cpuArchitecture: "x86_64"
    - openshiftVersion: "4.20"
      version: 9.6.20260107-0
      url: "https://mirror.openshift.com/pub/openshift-v4/aarch64/dependencies/rhcos/4.20/latest/rhcos-4.20.0-aarch64-live-iso.aarch64.iso"
      rootFSUrl: "https://mirror.openshift.com/pub/openshift-v4/aarch64/dependencies/rhcos/4.20/latest/rhcos-4.20.0-aarch64-live-rootfs.aarch64.img"
      cpuArchitecture: "aarch64"
EOF
~~~

~~~bash
$ oc create -f agent-service-config.yaml 
agentserviceconfig.agent-install.openshift.io/agent created
~~~

~~~bash
$ oc get pods -n multicluster-engine
NAME                                                   READY   STATUS    RESTARTS      AGE
agentinstalladmission-679cd54c5f-qjvfn                 1/1     Running   0             87s
agentinstalladmission-679cd54c5f-slj4s                 1/1     Running   0             87s
assisted-image-service-0                               1/1     Running   0             86s
assisted-service-587c875884-qcfb2                      2/2     Running   0             88s
cluster-curator-controller-7c66f8b67f-hbhkr            1/1     Running   0             24h
cluster-image-set-controller-6879c9fdf7-vhvsp          1/1     Running   0             24h
cluster-manager-847d499df7-kb5bx                       1/1     Running   0             24h
cluster-manager-847d499df7-w2sdj                       1/1     Running   0             24h
cluster-manager-847d499df7-z65kp                       1/1     Running   0             24h
cluster-proxy-addon-manager-86484759b9-mhgpg           1/1     Running   0             24h
cluster-proxy-addon-user-5fff4bbf8-57r7v               2/2     Running   0             24h
cluster-proxy-fbf4447f4-ch8p9                          1/1     Running   0             24h
clusterclaims-controller-dfcf6dcd4-b4p44               2/2     Running   0             24h
clusterlifecycle-state-metrics-v2-7c66dbd6f9-pslqq     1/1     Running   0             24h
console-mce-console-7dbbc66784-bb292                   1/1     Running   0             24h
discovery-operator-7997f54695-6mdct                    1/1     Running   0             24h
hcp-cli-download-5c4dfbfd6c-lgdhz                      1/1     Running   0             24h
hive-operator-6545b5986b-6pttn                         1/1     Running   0             24h
hypershift-addon-manager-64797b9868-h26wg              1/1     Running   0             24h
infrastructure-operator-5f9d89c69-k9b82                1/1     Running   1 (11h ago)   24h
managedcluster-import-controller-v2-75b55d65bd-4h8b4   1/1     Running   1 (11h ago)   24h
multicluster-engine-operator-6dd66fff8-gphcf           1/1     Running   0             24h
multicluster-engine-operator-6dd66fff8-tq6mx           1/1     Running   0             24h
ocm-controller-84964b45bb-h5hvs                        1/1     Running   0             24h
ocm-proxyserver-8cbffb748-mj5hx                        1/1     Running   0             24h
ocm-webhook-7d99759b8d-5dv9j                           1/1     Running   0             24h
provider-credential-controller-6f54b788b5-zm9bd        2/2     Running   0             24h
~~~

~~~bash
$ cat <<EOF >credentials.yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: hcp-adlink
  namespace: default
  labels:
    cluster.open-cluster-management.io/credentials: ""
    cluster.open-cluster-management.io/type: hostinventory
stringData:
  baseDomain: schmaustech.com
  pullSecret: PULL-SECRET    # Update with pull-secret 
  ssh-publickey: SSH-KEY     # Update with ssh-key
EOF
~~~

~~~bash
$ oc create -f credentials.yaml 
secret/hcp-adlink created
~~~

~~~bash
$ cat <<EOF >infrastructure-environment.yaml
kind: Secret
apiVersion: v1
metadata:
  name: pullsecret-hcp-adlink
  namespace: hcp-adlink
data:
  '.dockerconfigjson': 'PULL-SECRET-REDACTED'
type: 'kubernetes.io/dockerconfigjson'
---
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: hcp-adlink
  namespace: hcp-adlink
  labels:
    agentclusterinstalls.extensions.hive.openshift.io/location: Minneapolis
    networkType: dhcp
spec:
  agentLabels:
    'agentclusterinstalls.extensions.hive.openshift.io/location': Minneapolis
  pullSecretRef:
    name: pullsecret-hcp-adlink
  sshAuthorizedKey: SSH-KEY-REDACTED
  nmStateConfigLabelSelector:	
      matchLabels:	
        infraenvs.agent-install.openshift.io: hcp-adlink
  cpuArchitecture: x86_64
  osImageVersion: '4.20'
status:
  agentLabelSelector:
    matchLabels:
      'agentclusterinstalls.extensions.hive.openshift.io/location': Minneapolis
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: capi-provider-role
  namespace: hcp-adlink
rules:
  - verbs:
      - '*'
    apiGroups:
      - agent-install.openshift.io
    resources:
      - agents
EOF
~~~

~~~bash
$ oc create -f infrastructure-environment.yaml 
secret/pullsecret-hcp-adlink created
infraenv.agent-install.openshift.io/hcp-adlink created
role.rbac.authorization.k8s.io/capi-provider-role created
~~~
