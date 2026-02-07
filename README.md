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

With the custom resource file generated we can create it on the cluster.

~~~bash
$ oc create -f multicluster-engine-instance.yaml 
multiclusterengine.multicluster.openshift.io/multiclusterengine created
~~~

Once the multicluster engine is up and running we should see the following pods under the multicluster-engine namespace.

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

Next we need to patch the multicluster-engine to enable hosted control planes (aka hypershift).

~~~bash
$ oc patch mce multiclusterengine --type=merge -p   '{"spec":{"overrides":{"components":[{"name":"hypershift","enabled": true}]}}}'
multiclusterengine.multicluster.openshift.io/multiclusterengine patched
~~~

We can validate it's enabled with the following.

~~~bash
$ oc get managedclusteraddons -n local-cluster hypershift-addon
NAME               AVAILABLE   DEGRADED   PROGRESSING
hypershift-addon   True        False      False
~~~

We also need to create a provisioning configuration.

~~~bash
$ cat <<EOF >provisioning-config.yaml 
apiVersion: metal3.io/v1alpha1
kind: Provisioning
metadata:
  name: provisioning-configuration
spec:
  provisioningNetwork: "Disabled"
  watchAllNamespaces: true
EOF
~~~

Then create the provisioning configuration on the cluster.

~~~bash
$ oc create -f provisioning-config.yaml
provisioning.metal3.io/provisioning-configuration created
~~~

Now that hosted control planes are enabled we need to create a AgentServiceConfig custom resource file.

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
        storage: 15Gi
  filesystemStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi
  imageStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi
EOF
~~~

With the AgentServiceConfig custom resource file generated let's create it on the cluster.

~~~bash
$ oc create -f agent-service-config.yaml 
agentserviceconfig.agent-install.openshift.io/agent created
~~~

We can validate that the agent service is running by finding the assisted-image-service and assisted-service running under the multicluster-engine namespace.

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

Now that the multicluster engine is up and running we need to create a few secrets for our hosted cluster.  In this example our hosted cluster will be called hcp-adlink.   The first secret is for setting the base domain, pull-secret and ssh-key.

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

Let's create the key on the cluster.

~~~bash
$ oc create -f credentials.yaml 
secret/hcp-adlink created
~~~

Next we need a secret for our infrastructure environment.  The following is an example again where our cluster name is hcp-adlink. Also notice here that we are defining the CPU architecture here as aarch64 since our hosted workers will be arm64.

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
  cpuArchitecture: arm64
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

Once we have generated the custom resource file we can create it on the cluster.

~~~bash
$ oc create -f infrastructure-environment.yaml 
secret/pullsecret-hcp-adlink created
infraenv.agent-install.openshift.io/hcp-adlink created
role.rbac.authorization.k8s.io/capi-provider-role created
~~~

We can also see our infrastructure environment here.

~~~bash
$ oc get infraenv -n hcp-adlink
NAME         ISO CREATED AT
hcp-adlink   2026-02-07T15:59:38Z
~~~

This completes the initial configuration of multicluster engine.

## Install & Configuring Metallb Operator

Before we move forward with deploying a hosted control plane cluster we need to install the Metallb Operator on our cluster that will host the the hosted control plane.  The reason for this is Metallb will provide a loadbalancer and vip ipaddress for the api of our hosted cluster.   The first step here is to install the Metallb Operator using the following custom resource file.

~~~bash
$ cat <<EOF >metallb-operator.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: metallb-operator
  namespace: metallb-system
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metallb-operator-sub
  namespace: metallb-system
spec:
  channel: stable
  name: metallb-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

With the custom resource file generated we can create the resources on the cluster.

~~~bash
$ oc create -f metallb-operator.yaml
namespace/metallb-system created
operatorgroup.operators.coreos.com/metallb-operator created
subscription.operators.coreos.com/metallb-operator-sub created
~~~

Next we have to generate a MetalLB instance using the following custom resource file.

~~~bash
$ cat <<EOF >metallb-instance.yaml
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
EOF
~~~

With the custom resource file generated we can create the resource on the cluster.

~~~bash
$ oc create -f metallb-instance.yaml
metallb.metallb.io/metallb created
~~~

Finally we can check and see if all our MetalLB pods are up and running.

~~~bash
$ oc get pods -n metallb-system
NAME                                                   READY   STATUS    RESTARTS   AGE
controller-7f78f89f5f-hj4vb                            2/2     Running   0          28s
metallb-operator-controller-manager-84544fc95f-pfm89   1/1     Running   0          3m28s
metallb-operator-webhook-server-644c4c9758-5t6xm       1/1     Running   0          3m27s
speaker-55xt7                                          2/2     Running   0          28s
speaker-kclzj                                          2/2     Running   0          28s
speaker-mdjjn                                          2/2     Running   0          28s
~~~

If the pods are up and running we have two more steps we need to take.  The first is to generate a IPAddressPool for MetalLB so it knows where to get the ipaddresses for resources like our hosted control plane when they request it.  We can use the following custom resource file to accomplish that.

~~~bash
$ cat <<EOF >metallb-ipaddresspool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: hcp-network
  namespace: metallb-system
spec:
  addresses:
    - 192.168.0.170-192.168.0.172
  autoAssign: true
~~~

With the custom resource file generated we can create the resource on the cluster.

~~~bash
$ oc create -f metallb-ipaddresspool.yaml
ipaddresspool.metallb.io/hcp-network created
~~~

~~~bash
$ cat <<EOF >metallb-l2advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: advertise-hcp-network
  namespace: metallb-system
spec:
  ipAddressPools:
    - hcp-network
~~~

~~~bash
$ oc create -f metallb-l2advertisement.yaml
l2advertisement.metallb.io/advertise-hcp-network created
~~~

This completes the steps of configuration for MetalLB on the cluster that will host our hosted control plane.

## Deploying a Hosted Control Plane Cluster

In previous steps we went ahead and configured MultiCluster Engine Operator and Hosted Control Planes along with creating an infrastructure environment.   At this point we are almost ready to deploy a hosted control plane cluster but we first need to add some nodes to our infrastructure environment.   To do this we will first extract the minimal ISO from our infrastructure environment.

~~~bash
$ oc get infraenv -n hcp-adlink hcp-adlink -o jsonpath='{.status.isoDownloadURL}' | sed s/minimal-iso/full-iso/g | xargs curl -kLo ~/discovery-hcp-adlink.iso
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 98.3M  100 98.3M    0     0  10.8M      0  0:00:09  0:00:09 --:--:-- 10.8M

$ ls -lh ~/discovery-hcp-adlink.iso 
-rw-r--r--. 1 bschmaus bschmaus 99M Feb  7 10:08 /home/bschmaus/discovery-hcp-adlink.iso
~~~

We are not going to take that discovery-hcp-adlink.iso and boot it on a few of our Arm64 nodes.  After letting the nodes boot and waiting a few minutes we can see they have shown up under our agent pool in the hcp-adlink namespace.

~~~bash
$ oc get agent -n hcp-adlink
NAME                                   CLUSTER   APPROVED   ROLE          STAGE
0589f6a3-fb83-4f60-b4d4-3617c7023ca7             false      auto-assign   
5c9e6934-ea82-45b0-ab01-acb0626d86c5             false      auto-assign   
8c2920ce-6d30-4276-b51e-04ce22dcfae6             false      auto-assign  
~~~

Currently the nodes are not marked as approved so we need to approve them to make them usable in agent pool.

~~~bash
$ oc get agent -n hcp-adlink -ojson | jq -r '.items[] | select(.spec.approved==false) | .metadata.name'| xargs oc -n hcp-adlink patch -p '{"spec":{"approved":true}}' --type merge agent
agent.agent-install.openshift.io/0589f6a3-fb83-4f60-b4d4-3617c7023ca7 patched
agent.agent-install.openshift.io/5c9e6934-ea82-45b0-ab01-acb0626d86c5 patched
agent.agent-install.openshift.io/8c2920ce-6d30-4276-b51e-04ce22dcfae6 patched

$ oc get agent -n hcp-adlink
NAME                                   CLUSTER   APPROVED   ROLE          STAGE
0589f6a3-fb83-4f60-b4d4-3617c7023ca7             true       auto-assign   
5c9e6934-ea82-45b0-ab01-acb0626d86c5             true       auto-assign   
8c2920ce-6d30-4276-b51e-04ce22dcfae6             true       auto-assign   
~~~

~~~bash
$ cat <<EOF >hosted-cluster-deployment.yaml 
---
apiVersion: hypershift.openshift.io/v1beta1
kind: HostedCluster
metadata:
  name: 'hcp-adlink'
  namespace: 'hcp-adlink'
  labels:
    "cluster.open-cluster-management.io/clusterset": 'default'
spec:
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.20.13-multi
  pullSecret:
    name: pullsecret-cluster-hcp-adlink
  sshKey:
    name: sshkey-cluster-hcp-adlink
  networking:
    clusterNetwork:
      - cidr: 10.132.0.0/14
    serviceNetwork:
      - cidr: 172.31.0.0/16
    networkType: OVNKubernetes
  controllerAvailabilityPolicy: SingleReplica
  infrastructureAvailabilityPolicy: SingleReplica
  olmCatalogPlacement: management
  platform:
    type: Agent
    agent:
      agentNamespace: 'hcp-adlink'
  infraID: 'hcp-adlink'
  dns:
    baseDomain: 'schmaustech.com'
  services:
  - service: APIServer
    servicePublishingStrategy:
      type: LoadBalancer
  - service: OAuthServer
    servicePublishingStrategy:
      type: Route
  - service: OIDC
    servicePublishingStrategy:
      type: Route
  - service: Konnectivity
    servicePublishingStrategy:
      type: Route
  - service: Ignition
    servicePublishingStrategy:
      type: Route
---
apiVersion: v1
kind: Secret
metadata:
  name: pullsecret-cluster-hcp-adlink
  namespace: hcp-adlink
data:
  '.dockerconfigjson': <REDACTED PULL SECRET>
type: kubernetes.io/dockerconfigjson
---
apiVersion: v1
kind: Secret
metadata:
  name: sshkey-cluster-hcp-adlink
  namespace: 'hcp-adlink'
stringData:
  id_rsa.pub: <REDACTED SSH-KEY>
---
apiVersion: hypershift.openshift.io/v1beta1
kind: NodePool
metadata:
  name: 'nodepool-hcp-adlink-1'
  namespace: 'hcp-adlink'
spec:
  clusterName: 'hcp-adlink'
  replicas: 3
  management:
    autoRepair: false
    upgradeType: InPlace
  platform:
    type: Agent
    agent:
      agentLabelSelector:
        matchLabels: {}
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.20.13-multi
---
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  annotations:
    import.open-cluster-management.io/hosting-cluster-name: local-cluster 
    import.open-cluster-management.io/klusterlet-deploy-mode: Hosted
    open-cluster-management/created-via: hypershift
  labels:
    cloud: BareMetal
    vendor: OpenShift
    name: 'hcp-adlink'
    cluster.open-cluster-management.io/clusterset: 'default'
  name: 'hcp-adlink'
spec:
  hubAcceptsClient: true
---
~~~

~~~bash
$ oc create -f hosted-cluster-deployment.yaml 
hostedcluster.hypershift.openshift.io/hcp-adlink created
secret/pullsecret-cluster-hcp-adlink created
secret/sshkey-cluster-hcp-adlink created
nodepool.hypershift.openshift.io/nodepool-hcp-adlink-1 created
managedcluster.cluster.open-cluster-management.io/hcp-adlink created
~~~

~~~bash
$ oc get hostedcluster -n hcp-adlink
NAME         VERSION   KUBECONFIG   PROGRESS   AVAILABLE   PROGRESSING   MESSAGE
hcp-adlink                          Partial    False       False         Cluster infrastructure is still provisioning

$ oc get hostedcluster -n hcp-adlink
NAME         VERSION   KUBECONFIG   PROGRESS   AVAILABLE   PROGRESSING   MESSAGE
hcp-adlink                          Partial    False       False         Waiting for hosted control plane kubeconfig to be created

$ oc get hostedcluster -n hcp-adlink
NAME         VERSION   KUBECONFIG                    PROGRESS   AVAILABLE   PROGRESSING   MESSAGE
hcp-adlink             hcp-adlink-admin-kubeconfig   Partial    True        False         The hosted control plane is available
~~~

~~~bash
$ oc get nodepool nodepool-hcp-adlink-1 -n hcp-adlink
NAME                    CLUSTER      DESIRED NODES   CURRENT NODES   AUTOSCALING   AUTOREPAIR   VERSION   UPDATINGVERSION   UPDATINGCONFIG   MESSAGE
nodepool-hcp-adlink-1   hcp-adlink   3                               False         False        4.20.13   False             False            Scaling up MachineSet to 3 replicas (actual 0)
~~~

~~~bash
$ oc get agent -n hcp-adlink
NAME                                   CLUSTER      APPROVED   ROLE          STAGE
0589f6a3-fb83-4f60-b4d4-3617c7023ca7   hcp-adlink   true       auto-assign   
5c9e6934-ea82-45b0-ab01-acb0626d86c5   hcp-adlink   true       auto-assign   
8c2920ce-6d30-4276-b51e-04ce22dcfae6   hcp-adlink   true       auto-assign

$ oc get agent -n hcp-adlink
NAME                                   CLUSTER      APPROVED   ROLE     STAGE
0589f6a3-fb83-4f60-b4d4-3617c7023ca7   hcp-adlink   true       worker   Rebooting
5c9e6934-ea82-45b0-ab01-acb0626d86c5   hcp-adlink   true       worker   Rebooting
8c2920ce-6d30-4276-b51e-04ce22dcfae6   hcp-adlink   true       worker   Rebooting

$ oc get agent -n hcp-adlink
NAME                                   CLUSTER      APPROVED   ROLE     STAGE
0589f6a3-fb83-4f60-b4d4-3617c7023ca7   hcp-adlink   true       worker   Joined
5c9e6934-ea82-45b0-ab01-acb0626d86c5   hcp-adlink   true       worker   Joined
8c2920ce-6d30-4276-b51e-04ce22dcfae6   hcp-adlink   true       worker   Joined

$ oc get agent -n hcp-adlink
NAME                                   CLUSTER      APPROVED   ROLE     STAGE
0589f6a3-fb83-4f60-b4d4-3617c7023ca7   hcp-adlink   true       worker   Done
5c9e6934-ea82-45b0-ab01-acb0626d86c5   hcp-adlink   true       worker   Done
8c2920ce-6d30-4276-b51e-04ce22dcfae6   hcp-adlink   true       worker   Done
~~~

~~~bash
$ oc get nodepool nodepool-hcp-adlink-1 -n hcp-adlink
NAME                    CLUSTER      DESIRED NODES   CURRENT NODES   AUTOSCALING   AUTOREPAIR   VERSION   UPDATINGVERSION   UPDATINGCONFIG   MESSAGE
nodepool-hcp-adlink-1   hcp-adlink   3               3               False         False        4.20.13   False             False 
~~~

~~~bash
$ oc get hostedcluster -n hcp-adlink
NAME         VERSION   KUBECONFIG                    PROGRESS   AVAILABLE   PROGRESSING   MESSAGE
hcp-adlink             hcp-adlink-admin-kubeconfig   Partial    True        False         The hosted control plane is available
~~~

~~~bash
oc get secret -n hcp-adlink hcp-adlink-admin-kubeconfig  -ojsonpath='{.data.kubeconfig}'| base64 -d > ~/kubeconfig-hcp-adlink
~~~

~~~bash
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
console                                    4.20.13   False       False         True       88m     RouteHealthAvailable: failed to GET route (https://console-openshift-console.apps.hcp-adlink.schmaustech.com): Get "https://console-openshift-console.apps.hcp-adlink.schmaustech.com": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
csi-snapshot-controller                    4.20.13   True        False         False      112m    
dns                                        4.20.13   True        False         False      87m     
image-registry                             4.20.13   True        False         False      88m     
ingress                                    4.20.13   True        False         True       111m    The "default" ingress controller reports Degraded=True: DegradedConditions: One or more other status conditions indicate a degraded state: CanaryChecksSucceeding=False (CanaryChecksRepetitiveFailures: Canary route checks for the default ingress controller are failing. Last 2 error messages:...
insights                                   4.20.13   True        False         False      89m     
kube-apiserver                             4.20.13   True        False         False      112m    
kube-controller-manager                    4.20.13   True        False         False      112m    
kube-scheduler                             4.20.13   True        False         False      112m    
kube-storage-version-migrator              4.20.13   True        False         False      89m     
monitoring                                 4.20.13   True        False         False      80m     
network                                    4.20.13   True        False         False      90m     
node-tuning                                4.20.13   True        False         False      96m     
openshift-apiserver                        4.20.13   True        False         False      112m    
openshift-controller-manager               4.20.13   True        False         False      112m    
openshift-samples                          4.20.13   True        False         False      87m     
operator-lifecycle-manager                 4.20.13   True        False         False      112m    
operator-lifecycle-manager-catalog         4.20.13   True        False         False      112m    
operator-lifecycle-manager-packageserver   4.20.13   True        False         False      112m    
service-ca                                 4.20.13   True        False         False      89m     
storage                                    4.20.13   True        False         False      112m    
~~~

~~~bash
$ cat <<EOF >metallb-operator.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: metallb-operator
  namespace: metallb-system
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metallb-operator-sub
  namespace: metallb-system
spec:
  channel: stable
  name: metallb-operator
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
EOF
~~~

~~~bash
$ oc create -f metallb-operator.yaml
namespace/metallb-system created
operatorgroup.operators.coreos.com/metallb-operator created
subscription.operators.coreos.com/metallb-operator-sub created
~~~

~~~bash
$ cat <<EOF >metallb-instance.yaml
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
EOF
~~~

~~~bash
$ oc create -f metallb-instance.yaml
metallb.metallb.io/metallb created
~~~

~~~bash
$ oc get pods -n metallb-system
NAME                                                  READY   STATUS    RESTARTS   AGE
controller-7f78f89f5f-94m87                           2/2     Running   0          29s
metallb-operator-controller-manager-76f58797d-69jdm   1/1     Running   0          92s
metallb-operator-webhook-server-6d96484469-5z87l      1/1     Running   0          90s
speaker-72wfx                                         1/2     Running   0          29s
speaker-8h9vh                                         2/2     Running   0          29s
speaker-t455x                                         2/2     Running   0          29s
~~~

~~~bash
$ cat <<EOF >metallb-ipaddresspool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: hcp-network
  namespace: metallb-system
spec:
  addresses:
    - 192.168.0.173-192.168.0.175
  autoAssign: true
EOF
~~~

~~~bash
$ oc create -f metallb-ipaddresspool.yaml
ipaddresspool.metallb.io/hcp-network created
~~~

~~~bash
$ cat <<EOF >metallb-l2advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: advertise-hcp-network
  namespace: metallb-system
spec:
  ipAddressPools:
    - hcp-network
EOF
~~~

~~~bash
$ oc create -f metallb-l2advertisement.yaml
l2advertisement.metallb.io/advertise-hcp-network created
~~~

~~~bash
$ oc get ipaddresspool -n metallb-system -o yaml
apiVersion: v1
items:
- apiVersion: metallb.io/v1beta1
  kind: IPAddressPool
  metadata:
    creationTimestamp: "2026-02-07T20:03:50Z"
    generation: 1
    name: hcp-network
    namespace: metallb-system
    resourceVersion: "21540"
    uid: 6bf55fee-d634-4789-a19e-8ce505ba8efb
  spec:
    addresses:
    - 192.168.0.173-192.168.0.175
    autoAssign: true
    avoidBuggyIPs: false
  status:
    assignedIPv4: 0
    assignedIPv6: 0
    availableIPv4: 3
    availableIPv6: 0
kind: List
metadata:
  resourceVersion: ""
~~~

~~~bash
$ cat <<EOF >hcp-adlink-metallb-ingress.yaml
kind: Service
apiVersion: v1
metadata:
  annotations:
   metallb.io/address-pool: hcp-network
  name: metallb-ingress
  namespace: openshift-ingress
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
  selector:
    ingresscontroller.operator.openshift.io/deployment-ingresscontroller: default
  type: LoadBalancer
EOF
~~~

~~~bash
$ oc get service -n openshift-ingress
NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
metallb-ingress           LoadBalancer   172.31.54.206   192.168.0.173   80:31492/TCP,443:30929/TCP   9s
router-internal-default   ClusterIP      172.31.142.3    <none>          80/TCP,443/TCP,1936/TCP      119m
~~~

~~~bash
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
console                                    4.20.13   True        False         False      83s     
csi-snapshot-controller                    4.20.13   True        False         False      121m    
dns                                        4.20.13   True        False         False      96m     
image-registry                             4.20.13   True        False         False      97m     
ingress                                    4.20.13   True        False         False      120m    
insights                                   4.20.13   True        False         False      98m     
kube-apiserver                             4.20.13   True        False         False      121m    
kube-controller-manager                    4.20.13   True        False         False      121m    
kube-scheduler                             4.20.13   True        False         False      121m    
kube-storage-version-migrator              4.20.13   True        False         False      97m     
monitoring                                 4.20.13   True        False         False      89m     
network                                    4.20.13   True        False         False      98m     
node-tuning                                4.20.13   True        False         False      105m    
openshift-apiserver                        4.20.13   True        False         False      121m    
openshift-controller-manager               4.20.13   True        False         False      121m    
openshift-samples                          4.20.13   True        False         False      96m     
operator-lifecycle-manager                 4.20.13   True        False         False      121m    
operator-lifecycle-manager-catalog         4.20.13   True        False         False      121m    
operator-lifecycle-manager-packageserver   4.20.13   True        False         False      121m    
service-ca                                 4.20.13   True        False         False      98m     
storage                                    4.20.13   True        False         False      121m 
~~~

~~~bash
$ telnet console-openshift-console.apps.hcp-adlink.schmaustech.com 443
Trying 192.168.0.173...
Connected to console-openshift-console.apps.hcp-adlink.schmaustech.com.
Escape character is '^]'.
^]
telnet> quit
Connection closed.
~~~

