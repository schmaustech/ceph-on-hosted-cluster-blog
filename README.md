# **Workloads on Bare Metal Hosted Clusters Deployed From Red Hat Advanced Cluster Management for Kubernetes**

<img src="high-level-overview.png" style="width: 1000px;" border=0/>

In a previous blog on [How to Build Bare Metal Hosted Clusters on Red Hat Advanced Cluster Management for Kubernetes](https://cloud.redhat.com/blog/how-to-build-bare-metal-hosted-clusters-on-red-hat-advanced-cluster-management-for-kubernetes) I discussed how one could deploy a hosted cluster.  The blog outlined the benefits of running hosted clusters which included minimal time to deploy and cost savings due to the control plane running on an existing OpenShift cluster.  I further demonstrated how to build out the environment and validate that the installation was completed sucessfully.  Today however I want to move onto the day two activities like running workloads on that hosted cluster which is exactly what we will cover in this blog.   

## Lab Environment

First lets review the lab environment so we are familiar with how the hosted cluster was deployed.  Looking back we originally had a 3 node compact  Red Hat Advanced Cluster Management for Kubernetes 2.6 hub cluster running on OpenShift 4.10.26 called kni20.  This hub cluster has since been upgraded to OpenShift 4.11.3.  We used the hub cluster to deploy a hosted cluster running OpenShift 4.11.2 called kni21 where the control plane is running as containers on our hub cluster and then we also have 3 bare metal worker nodes for our workloads.  The high level architecture looks like the image below:

<img src="hosted1.jpeg" style="width: 800px;" border=0/>

The kni21 worker nodes contain a second disk which we will make use of for our workloads from a storage perspective.  Further our DNS records for our kni21 environment look like the following since we were using the nodeport method for exposing our api and ingress routes:

~~~bash
Name:	api.kni21.schmaustech.com
Address: 192.168.0.211

Name:	*.apps.kni21.schmaustech.com
Address: 192.168.0.116
Name:	*.apps.kni21.schmaustech.com
Address: 192.168.0.118
Name:	*.apps.kni21.schmaustech.com
Address: 192.168.0.117
~~~

Now that we have an idea of how the environment is setup lets turn our attention to running workloads.

## The Workloads

When it comes to deploying workloads on a hosted cluster it really should not be any different then when deploying on a standard baremetal OpenShift cluster.   The same methods whether using cli or UI apply in installing and configuring various operators and applications.  Those operators and applications will get scheduled on any schedulable worker node in the cluster.  For the purpose of this blog I will be deploying the following workloads and supporting operators:

 * Local Storage Operator
 * OpenShift Data Foundation
 * OpenShift Containerized Virtualization
 * Launching a virtual machine

Before we begin however we should get access and validate our hosted cluster is ready to accept the workloads defined above.

First lets validate the hosted cluster is ready and online.  We can first check the status of the hosted cluster object.

~~~bash
$ oc get hostedcluster -n kni21
NAME    VERSION   KUBECONFIG               PROGRESS    AVAILABLE   PROGRESSING   MESSAGE
kni21   4.11.2    kni21-admin-kubeconfig   Completed   True        False         The hosted control plane is available
~~~

We can also validate the nodepool to confirm the desired worker node count is present:

~~~bash
$ oc get nodepool -n kni21
NAME               CLUSTER   DESIRED NODES   CURRENT NODES   AUTOSCALING   AUTOREPAIR   VERSION   UPDATINGVERSION   UPDATINGCONFIG   MESSAGE
nodepool-kni21-1   kni21     3               3               False         False        4.11.2  
~~~

And finally we can now extract the kubeconfig for the kni21 cluster so we can run commands against the hosted cluster.

~~~bash
$ oc extract -n kni21 secret/kni21-admin-kubeconfig --to=- > kubeconfig-kni21
# kubeconfig
~~~

Now lets go ahead and set our KUBECONFIG variable to the kubeconfig we extracted and view the nodes.

~~~bash
$ export KUBECONFIG=/home/bschmaus/kubeconfig-kni21
$ oc get nodes
NAME                            STATUS   ROLES    AGE   VERSION
asus3-vm1.kni.schmaustech.com   Ready    worker   75m   v1.24.0+b62823b
asus3-vm2.kni.schmaustech.com   Ready    worker   75m   v1.24.0+b62823b
asus3-vm3.kni.schmaustech.com   Ready    worker   74m   v1.24.0+b62823b
~~~

We can see from the above output that all our worker nodes for the hosted cluster kni21 are in a ready state.   I should point out here that we do not see any control plane nodes listed and this is because there are no control plane nodes only control plane service pods which reside on our hub cluster.

Let's move onto installing and configuring our workloads.

## Deploying Local Storage Operator

Now that we know our hosted cluster is ready to consume workloads lets get started by installing the Local Storage Operator.  When we are finished we should have the results similar to what is in the following diagram in that a local storage PV will be created for every sdb block device.

<img src="hosted2.jpeg" style="width: 800px;" border=0/>

I already know that in each of my worker nodes I have a secondary 120GB block device called sdb so I will skip using oc debug to check for the block devices.   Next I will go ahead and label my worker nodes for storage.

~~~bash
$ oc label nodes asus3-vm1.kni.schmaustech.com cluster.ocs.openshift.io/openshift-storage=''
node/asus3-vm1.kni.schmaustech.com labeled
$ oc label nodes asus3-vm2.kni.schmaustech.com cluster.ocs.openshift.io/openshift-storage=''
node/asus3-vm2.kni.schmaustech.com labeled
$ oc label nodes asus3-vm3.kni.schmaustech.com cluster.ocs.openshift.io/openshift-storage=''
node/asus3-vm3.kni.schmaustech.com labeled
~~~

Now let's confirm the label changes before we proceed:

~~~bash
$ oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
NAME                            STATUS   ROLES    AGE   VERSION
asus3-vm1.kni.schmaustech.com   Ready    worker   83m   v1.24.0+b62823b
asus3-vm2.kni.schmaustech.com   Ready    worker   83m   v1.24.0+b62823b
asus3-vm3.kni.schmaustech.com   Ready    worker   82m   v1.24.0+b62823b
~~~

Now we can proceed by installing the Local Storage Operator which will be used by OpenShift Data Foundation.  The first thing we need to do is create the namespace of it by creating the custom resource yaml and then applying it to the hosted cluster.

~~~bash
cat << EOF > ~/openshift-local-storage-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-local-storage
spec: {}
EOF

$ oc create -f ~/openshift-local-storage-namespace.yaml
namespace/openshift-local-storage created
~~~

Next we can create the storage group customer resource yaml and then apply that as well to the hosted cluster.

~~~bash
cat << EOF > ~/openshift-local-storage-group.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: local-operator-group
  namespace: openshift-local-storage
spec:
  targetNamespaces:
  - openshift-local-storage
EOF

oc create -f ~/openshift-local-storage-group.yaml
operatorgroup.operators.coreos.com/local-operator-group created
~~~

Now we can proceed to create a subscription for the Local Storage Operator and apply that to the hosted cluster.

~~~bash
$ cat << EOF > ~/openshift-local-storage-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: local-storage-operator
  namespace: openshift-local-storage
spec:
  channel: "stable"
  installPlanApproval: Automatic
  name: local-storage-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

$ oc create -f ~/openshift-local-storage-subscription.yaml
subscription.operators.coreos.com/local-storage-operator created
~~~

After a few minutes we should be able to see the operator running.

~~~bash
$ oc get pods -n openshift-local-storage
NAME                                      READY   STATUS    RESTARTS   AGE
local-storage-operator-864768c8cb-slgc2   1/1     Running   0          60s
~~~

Now that we have the local storage operator installed lets make a LocalVolume storage definition file that will use the disk device in each node. We can  see that this is set to create a local volume on every host from the block device sdb where the selector key matches cluster.ocs.openshift.io/openshift-storage. If we had additional devices on the worker nodes for example: sdd and sde, we would just list those below the devicePaths to also be incorporated into our configuration.

~~~bash
$ cat << EOF > ~/local-storage.yaml
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/sdb
EOF
~~~

Now we can go ahead and create the assets for this local-storage configuration using the local-storage.yaml we created above.

~~~bash
$ oc create -f ~/local-storage.yaml
localvolume.local.storage.openshift.io/local-block created
~~~

After a few minutes we should see a diskmaker manager pod running on each worker node in our hosted cluster.

~~~bash
$ oc -n openshift-local-storage get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE                            NOMINATED NODE   READINESS GATES
diskmaker-manager-lmzs8                   2/2     Running   0          32m   10.134.0.19   asus3-vm1.kni.schmaustech.com   <none>           <none>
diskmaker-manager-qmw5t                   2/2     Running   0          32m   10.133.0.20   asus3-vm3.kni.schmaustech.com   <none>           <none>
diskmaker-manager-x8qgz                   2/2     Running   0          32m   10.132.0.32   asus3-vm2.kni.schmaustech.com   <none>           <none>
local-storage-operator-864768c8cb-slgc2   1/1     Running   0          36m   10.133.0.19   asus3-vm3.kni.schmaustech.com   <none>           <none>
~~~

We can also see that three PVs were also created one for each worker node reflecting the use of the sdb device on each worker node in the hosted cluster.

~~~bash
$ oc get pv -o wide
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE   VOLUMEMODE
local-pv-3055d2e7   120Gi      RWO            Delete           Available           localblock              20m   Block
local-pv-978f0e97   120Gi      RWO            Delete           Available           localblock              20m   Block
local-pv-ceca062c   120Gi      RWO            Delete           Available           localblock              20m   Block
~~~

And finally a storageclass was created for the local-storage asset we created.

~~~bash
$ oc get sc
NAME         PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  22m
~~~

We have now completed installing and configuring the Local Storage Operator.   We can now move onto our next layer of our workload stack.

## Deploying OpenShift Data Foundation

At this point we have now completed the prerequisites of installing the Local Storage Operator. We can now turn our attention to installing OpenShift Data Foundation which will consume those local storage PVs and leverage them in a storage cluster which will provide block, object and file.  The following diagram depicts a high level overview of where those storage cluster components reside.

<img src="hosted3.jpeg" style="width: 800px;" border=0/>

To get started with the installation we need to go ahead and create the openshift-storage namespace.

~~~bash
$ cat << EOF > ~/openshift-storage-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    openshift.io/cluster-monitoring: "true"
  name: openshift-storage
spec: {}
EOF
~~~~

Next we will create the namespace from the customer resource yaml we created above.

~~~bash
$ oc create -f ~/openshift-storage-namespace.yaml
namespace/openshift-storage created
~~~

With the namespace created we need to create and operator group and a subscription just like we did with the local storage operator.  However this time we will create both from the same resource yaml file.

~~~bash
$ cat << EOF > ~/openshift-storage-subscription.yaml
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
  - openshift-storage
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ocs-operator
  namespace: openshift-storage
spec:
  channel: "stable-4.11"
  installPlanApproval: Automatic
  name: odf-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

Lets go ahead and create the operator group and subscription from the resource yaml file we created:

~~~bash
$ oc create -f ~/openshift-storage-subscription.yaml
operatorgroup.operators.coreos.com/openshift-storage-operatorgroup created
subscription.operators.coreos.com/ocs-operator created
~~~

With the namespace, openshift-storage operator group and subscription created we should in a few minutes have the required running pods for the OpenShift Data Foundation Operator in the environment.

~~~bash
$ oc get pods -n openshift-storage
NAME                                               READY   STATUS    RESTARTS   AGE
csi-addons-controller-manager-db678484c-h8lc6      2/2     Running   0          2m45s
noobaa-operator-67d4d55688-ztnf7                   1/1     Running   0          3m5s
ocs-metrics-exporter-977854c58-4pjxb               1/1     Running   0          84s
ocs-operator-678d6b5d89-6k8mj                      1/1     Running   0          2m51s
odf-console-54f74bc899-2gtb6                       1/1     Running   0          3m2s
odf-operator-controller-manager-65d8c775c4-k4w6h   2/2     Running   0          3m2s
rook-ceph-operator-7b55877858-x5q77                1/1     Running   0          2m50s
~~~

Now that we know the operator is deployed (and the associated pods are running) we can proceed to creating a hyperconverged OpenShift Data Foundation cluster.  To do this we first need to create a storage cluster yaml file that will allow us to consume each of the pvs per worker node that we configured via the Local Storage Operator.  We will also configure cpu, memory, replicas and which components the operator will manage in this file.

~~~bash
$ cat << EOF > ~/openshift-storage-cluster.yaml
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  manageNodes: false
  resources:
    mds:
      limits:
        cpu: "3"
        memory: "6Gi"
      requests:
        cpu: "3"
        memory: "6Gi"
  monDataDirHostPath: /var/lib/rook
  managedResources:
    cephBlockPools:
      reconcileStrategy: manage
    cephFilesystems:
      reconcileStrategy: manage
    cephObjectStoreUsers:
      reconcileStrategy: manage
    cephObjectStores:
      reconcileStrategy: manage
    snapshotClasses:
      reconcileStrategy: manage
    storageClasses:
      reconcileStrategy: manage
  multiCloudGateway:
    reconcileStrategy: manage
  storageDeviceSets:
  - count: 1
    dataPVCTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: "120Gi"
        storageClassName: localblock
        volumeMode: Block
    name: ocs-deviceset
    placement: {}
    portable: false
    replica: 3
    resources:
      limits:
        cpu: "2"
        memory: "5Gi"
      requests:
        cpu: "2"
        memory: "5Gi"
EOF
~~~

If we look at the above storage cluster yaml the key things that stand out are the count, the storage size, storageclass and replica.  Because we have 2 pvs on each worker that we want included in our storage cluster from the local storage storage class we set the replica to 3 and the count to 2.   We also specify that we want to use 100Gi pvs from the localblock storage class.  We can also note the resource limits on CPU and memory to ensure that the storage cluster does not use all the resources on the worker nodes.

With the storage cluster yaml created lets go ahead and create the storage cluster:

~~~bash
$ oc create -f ~/openshift-storage-cluster.yaml
storagecluster.ocs.openshift.io/ocs-storagecluster created
~~~

If we run an oc get pods for the openshift-storage namespace we can see pods are starting to create to build out the storage cluster.  If one wanted to watch this continuously we could throw a watch command in front of the oc command.   It will take a few minutes to instantiate the cluster nevertheless.

Finally after about 5 minutes we can see all the pods that have been generated to deploy the OCS storage cluster:

~~~bash
$ oc get pods -n openshift-storage
NAME                                                              READY   STATUS      RESTARTS      AGE
csi-addons-controller-manager-db678484c-h8lc6                     2/2     Running     2 (14m ago)   29m
csi-cephfsplugin-mgf92                                            3/3     Running     0             7m41s
csi-cephfsplugin-provisioner-7bdf9d685c-l28x5                     6/6     Running     0             7m41s
csi-cephfsplugin-provisioner-7bdf9d685c-tqdvw                     6/6     Running     0             7m42s
csi-cephfsplugin-w26pp                                            3/3     Running     0             7m41s
csi-cephfsplugin-zgrph                                            3/3     Running     0             7m42s
csi-rbdplugin-provisioner-cc79dfc7b-rfhw4                         7/7     Running     0             7m42s
csi-rbdplugin-provisioner-cc79dfc7b-zn7fb                         7/7     Running     0             7m43s
csi-rbdplugin-qkfbv                                               4/4     Running     0             7m43s
csi-rbdplugin-txxr5                                               4/4     Running     0             7m43s
csi-rbdplugin-xktrt                                               4/4     Running     0             7m43s
noobaa-core-0                                                     1/1     Running     0             82s
noobaa-db-pg-0                                                    1/1     Running     0             85s
noobaa-operator-67d4d55688-ztnf7                                  1/1     Running     1 (80s ago)   29m
ocs-metrics-exporter-977854c58-4pjxb                              1/1     Running     0             27m
ocs-operator-678d6b5d89-6k8mj                                     1/1     Running     2 (14m ago)   29m
odf-console-54f74bc899-2gtb6                                      1/1     Running     0             29m
odf-operator-controller-manager-65d8c775c4-k4w6h                  2/2     Running     1 (14m ago)   29m
rook-ceph-crashcollector-asus3-vm1.kni.schmaustech.com-6776zjlw   1/1     Running     0             2m6s
rook-ceph-crashcollector-asus3-vm2.kni.schmaustech.com-7cc4pn4b   1/1     Running     0             2m10s
rook-ceph-crashcollector-asus3-vm3.kni.schmaustech.com-784jxfwt   1/1     Running     0             3m13s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-757db8fb5vh4v   2/2     Running     0             2m12s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-97cf869-mfj7f   2/2     Running     0             2m9s
rook-ceph-mgr-a-698c899c69-8dmxh                                  2/2     Running     0             3m32s
rook-ceph-mon-a-5895f8556b-2d2w6                                  2/2     Running     0             4m45s
rook-ceph-mon-b-676cdcb5bf-5b26q                                  2/2     Running     0             4m14s
rook-ceph-mon-c-6b66d85467-t2wjp                                  2/2     Running     0             3m53s
rook-ceph-operator-7b55877858-x5q77                               1/1     Running     0             29m
rook-ceph-osd-0-76d7855dbb-r9q9f                                  2/2     Running     0             2m39s
rook-ceph-osd-1-7f5db57655-xwgkn                                  2/2     Running     0             2m36s
rook-ceph-osd-2-7cf6f8bc64-h58gz                                  2/2     Running     0             2m35s
rook-ceph-osd-prepare-ocs-deviceset-0-data-06j5df-z75jf           0/1     Completed   0             2m57s
rook-ceph-osd-prepare-ocs-deviceset-1-data-0gf9tv-v8f2t           0/1     Completed   0             2m55s
rook-ceph-osd-prepare-ocs-deviceset-2-data-0vbtdz-5kxb8           0/1     Completed   0             2m54s
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-a-79bcfc7xgfst   2/2     Running     0             2m2s
~~~

We can also confirm this from the command line by issuing an oc get storageclass.  We should see 4 new storageclasses: one for block (rbd), two for object (rgw/nooba) and one for file(cephfs).  

~~~bash
$ oc get storageclass
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock                    kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  15h
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   95m
ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  101m
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   95m
openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate              false                  92m
~~~

Before we move on lets go ahead and set the rbd storageclass to the default.

~~~bash
$ oc patch storageclass  ocs-storagecluster-ceph-rbd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/ocs-storagecluster-ceph-rbd patched

$ oc get sc
NAME                                    PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock                              kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  15h
ocs-storagecluster-ceph-rbd (default)   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   97m
ocs-storagecluster-ceph-rgw             openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  103m
ocs-storagecluster-cephfs               openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   97m
openshift-storage.noobaa.io             openshift-storage.noobaa.io/obc         Delete          Immediate              false                  94m

~~~

At this point you have a fully functional OpenShift Container Storage cluster to be consumed by applications. You can optionally deploy the Ceph tools pod, where you can dive into some of the Ceph internals at your leisure:


## OpenShift Containerized Virtualization

