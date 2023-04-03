# ceph-toolbox
Notes to Openshift Data Foundation monitoring

Using the do370 course information from RedHat:

https://rol.redhat.com/rol/app/courses/do370-4.7/pages/ch01

Comparing Internal Mode to External Mode Deployments
Administrators can choose to deploy OpenShift Data Foundation entirely within an RHOCP cluster (internal), or to expose Red Hat Ceph Storage services running outside the cluster (external).



oc get storagecluster -n openshift-storage

oc get storageclasses -o \
  custom-columns='NAME:metadata.name,PROVISIONER:provisioner'

oc get all -n openshift-local-storage

DF status:

oc get all -n openshift-storage

List the disks available in one of the worker nodes:

oc debug node/worker01 -- lsblk --paths --nodeps


Persistent Volumes and Persistent Volume Claims

PVCs are Kubernetes objects to request storage PVs. A PVC can include specific details such as the size and the access mode (ReadWriteOnce, ReadOnlyMany or ReadWriteMany). 

Storage Classes Provided by Red Hat OpenShift Data Foundation
Red Hat OpenShift Data Foundation configures the following storage classes to provide data services that support different storage technologies.

ocs-storagecluster-ceph-rbd. Installed by the Rook-Ceph operator, this class supports block storage devices, primarily for database workloads. Examples include the Red Hat OpenShift Container Platform (OCP) logging and monitoring stack and PostgreSQL.

ocs-storagecluster-cephfs. Installed by the Rook-Ceph operator, this class provides shared and distributed file system data services, primarily used for software development, messaging, and data aggregation workloads. Examples include Jenkins build sources and artifacts, WordPress uploaded content, the RHOCP registry, and messaging using JBoss AMQ.

openshift-storage.noobaa.io. Installed by the NooBaa operator, this class provides the Multicloud Object Gateway (MCG) service. Ultimately, the MCG service provides multicloud object storage as an S3 API endpoint that can abstract the storage and retrieval of data from multiple cloud object stores.

ocs-storagecluster-ceph-rgw. Installed by the Rook-Ceph operator, this class provides on-premises object storage. The Ceph RADOS Gateway (Ceph RGW) object storage interface is a robust S3 API endpoint that scales to tens of petabytes and billions of objects, primarily targeting data-intensive applications. Examples include the storage and access of row, columnar, and semi-structured data with applications like Spark, Presto, Red  Hat AMQ Streams (Kafka), and machine learning frameworks like TensorFlow and Pytorch.

Configuring Monitoring to Use Red Hat OpenShift Data Foundation


Explaining the RHOCP Monitoring Stack Architecture
The RHOCP monitoring stack provides built-in monitoring and alerts for the core platform components. Available operations include querying metrics, reviewing dashboards, and managing alerting rules. The monitoring stack is integrated into the RHOCP web console.

Furthermore, administrators can also enable monitoring for user-defined projects after the cluster installation.

The monitoring stack provided by RHOCP is based on Prometheus. 

Configuring Persistent Storage for the RHOCP Monitoring Stack
By default, the monitoring data storage is set to ephemeral. All the metrics are lost when the pods are restarted or recreated.

You can configure persistent storage volumes (PV) to store the monitoring data. This allows organizations to preserve metrics for post-mortem analysis, big data, cost management, or infrastructure load statistics.

Red Hat recommends using local storage in production environments because of the high IO storage demands. This approach is compatible with using an OpenShift Data Foundation cluster in internal mode.

The monitoring stack provided by RHOCP is based on Prometheus. 

Guided Exercise: Configuring Monitoring to Use Red Hat OpenShift Data Foundation
In this exercise, you will configure Prometheus and Alertmanager to use storage provided by Red Hat OpenShift Data Foundation.

oc get storageclasses -o name


Verify the disk space in the emptyDir volume mounted in /prometheus.

oc exec -n openshift-monitoring \
  statefulset/prometheus-k8s -c prometheus -- df -h /prometheus


Verify the disk space in the emptyDir volume mounted in /alertmanager.

oc exec -n openshift-monitoring \
  statefulset/alertmanager-main -c alertmanager -- df -h /alertmanager

Verify that both Prometheus and Alertmanager use block storage from OpenShift Data Foundation.


Create the cluster-monitoring-config configuration map in the openshift-monitoring namespace.

Modify the metrics-storage.yml file.

Configure Prometheus to retain metrics information for seven days.

Configure each prometheus-k8s pod to request a 40 GB PVC from the ocs-storagecluster-ceph-rbd storage class.

Configure each alertmanager-main pod to request a 20 GB PVC from the ocs-storagecluster-ceph-rbd storage class.

metrics-storage.yaml

prometheusK8s:
  retention: 7d
  volumeClaimTemplate:
    spec:
      storageClassName: ocs-storagecluster-ceph-rbd
      resources:
        requests:
          storage: 40Gi
alertmanagerMain:
  volumeClaimTemplate:
    spec:
      storageClassName: ocs-storagecluster-ceph-rbd
      resources:
        requests:
          storage: 20Gi

oc create -n openshift-monitoring \
  configmap cluster-monitoring-config --from-file config.yaml=metrics-storage.yml




Monitor the redeployment of prometheus-k8s and alertmanager-main pods in the openshift-monitoring namespace.

watch oc get statefulsets \
  -n openshift-monitoring


Verify the PVCs that are used to save the monitoring data are created.

oc get persistentvolumeclaims \
  -n openshift-monitoring

Verify that the prometheus-k8s stateful set in the openshift-monitoring namespace mounts a block device to the /prometheus mount point. A file system starting with /dev/rbd indicates block storage. Also check that the volume size is now 40 GB.

oc exec -n openshift-monitoring \
  statefulset/prometheus-k8s -c prometheus -- df -h /prometheus

Verify that the alertmanager-main stateful set in the openshift-monitoring namespace mounts a block device to the /alertmanager mount point. A file system starting with /dev/rbd indicates block storage. Also check that the volume size is now 20 GB.

oc exec -n openshift-monitoring \
  statefulset/alertmanager-main -c alertmanager -- df -h /alertmanager


Discussing Ceph with OpenShift
Ceph is managed by the Rook-Ceph operator. The Local Storage Operator (LSO) creates persistent volumes that are passed to the Rook-Ceph operator to create the Ceph cluster.

oc get pv

Each Ceph element runs inside a pod.

oc get pods -n openshift-storage

Investigating a Ceph Cluster
There are many ways to verify the health of a Ceph cluster, such as by reviewing the cluster logs, by using Ceph client tools, or by using the must-gather tool. These are some of the common ways that you can examine cluster health and obtain troubleshooting information on a Ceph cluster managed by OpenShift Data Foundation.

Inspecting Ceph Components Logs
Reviewing logs is often a quick way to determine the source of a problem. A problem will typically present itself in the form of error messages. With Ceph, many problems are self-healing, but if the cluster does not self-heal you will likely see error messages near the end of the logs.

This section provides information on how to access each Ceph log.

OSD logs

oc get po -n openshift-storage|grep osd
rook-ceph-osd-0-8699dc5bf7-7g5jn                                  2/2     Running   4              47d
rook-ceph-osd-1-85687d4bb4-6xwdk                                  2/2     Running   4              47d
rook-ceph-osd-2-6f76cffbcb-fhpjl                                  2/2     Running   4              47d

e.g.

oc logs rook-ceph-osd-0-8699dc5bf7-7g5jn -n openshift-storage osd


Monitor logs

oc get po -n openshift-storage|grep rook-ceph-mon

oc logs rook-ceph-mon-a-85d95d786-gk7cm -n openshift-storage log-collector


Rook Operator logs

oc get po -n openshift-storage|grep rook-ceph-oper

oc logs rook-ceph-operator-6f4568cf4c-hrzvc -n openshift-storage


CephFS file storage logs

oc get po -n openshift-storage|grep csi-cephfsplugin

oc logs csi-cephfsplugin-5566k -n openshift-storage -c csi-cephfsplugin


RBD plug-in block storage logs

oc get po -n openshift-storage|grep csi-rbdplugin

oc logs csi-rbdplugin-4x6wv -n openshift-storage -c csi-rbdplugin


PVC-specific logs

Logs for persistent volume claims (PVC) are handled by a designated pod. There is a pod in charge of each filesystem or block plug-in worker in the Ceph cluster.



Using Ceph Built-in Tools for Collecting Cluster Information

Ceph provides the built in tool must-gather and the Ceph CLI. These tools provide a quick way to obtain information and logs for review.

To use the Ceph CLI, enter the Rook-Ceph operator and use the /var/lib/rook/openshift-storage/openshift-storage.config configuration file to access the cluster:

oc get po -n openshift-storage|grep rook-ceph-operator
rook-ceph-operator-6f4568cf4c-hrzvc                               1/1     Running   2              47d

oc exec -ti \
  pod/rook-ceph-operator-6f4568cf4c-hrzvc -n openshift-storage \
  -c rook-ceph-operator -- /bin/bash

Cluster health

ceph -c /var/lib/rook/openshift-storage/openshift-storage.config health

Cluster status

ceph -c /var/lib/rook/openshift-storage/openshift-storage.config -s

Storage usage

ceph -c /var/lib/rook/openshift-storage/openshift-storage.config df


Using the must-gather Tool
Use the must-gather tool for collecting log files and diagnostic information about your cluster.

oc adm must-gather \
  --image=registry.redhat.io/ocs4/ocs-must-gather-rhel8:v4.7 \
  --dest-dir=must-gather


test dashboard:

oc rsh monitoring-ge-pod

while [ 1 ]; do dd if=/dev/zero of=/mnt/bigfile count=1 bs=100M; done

Extending Application Storage for Red Hat OpenShift Data Foundation
Objectives
After completing this section, you should be able to detect when a storage volume is close to full and expand the volume size without disrupting an application.

Persistent volume claims (PVC) can be extended to enable applications to expand their storage capacity. A PVC can be expanded only if its storage class has the allowVolumeExpansion attribute set to true which is the default.

oc describe sc managed-nfs-storage|grep -i allowVolumeExpansion
AllowVolumeExpansion:  <unset>

oc describe sc ocs-storagecluster-cephfs|grep -i allowVolumeExpansion
AllowVolumeExpansion:  True

oc describe sc ocs-storagecluster-ceph-rbd|grep -i allowVolumeExpansion
AllowVolumeExpansion:  True

oc exec pod/nginx-58fc88c76d-8rwrb -- \
  df -h | grep /space

oc exec ibm-nginx-6c6db57c66-pkc9w -n cpd-instance -- df -h |grep /user-home

172.30.255.69:6789,172.30.3.220:6789,172.30.130.49:6789:/volumes/csi/csi-vol-254247dc-7d28-11ed-b27d-0a580a800408/d8c903c6-b4cf-47c0-a666-e585ad448ea1   10G  256M  9.8G   3% /user-home

oc describe po ibm-nginx-6c6db57c66-pkc9w -n cpd-instance |grep -i pvc
    ClaimName:  user-home-pvc

oc get pvc -A |grep user-home-pvc
cpd-instance               user-home-pvc                                  Bound    pvc-a1db39d5-d3df-4744-81eb-7ff45137027f   10Gi       RWX            ocs-storagecluster-cephfs     45d

The CAPACITY field shows the storage assigned to the PVC: in this case 10Gi

PVC size can be increased by editing the resource:

oc edit pvc/user-home-pvc -n cpd-instance

change to 11Gi

spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 11Gi

oc get pvc -A |grep user-home-pvc
cpd-instance               user-home-pvc                                  Bound    pvc-a1db39d5-d3df-4744-81eb-7ff45137027f   11Gi       RWX            ocs-storagecluster-cephfs     45d

The oc patch command can also be used to increase the size of the PVC:

oc patch pvc/user-home-pvc -n cpd-instance \
  -p '{"spec":{"resources":{"requests": {"storage": "12Gi"}}}}'

oc get pvc -A |grep user-home-pvc
cpd-instance               user-home-pvc                                  Bound    pvc-a1db39d5-d3df-4744-81eb-7ff45137027f   12Gi       RWX            ocs-storagecluster-cephfs     45d


oc exec ibm-nginx-6c6db57c66-pkc9w -n cpd-instance -- df -h |grep /user-home


172.30.255.69:6789,172.30.3.220:6789,172.30.130.49:6789:/volumes/csi/csi-vol-254247dc-7d28-11ed-b27d-0a580a800408/d8c903c6-b4cf-47c0-a666-e585ad448ea1   12G  256M   12G   3% /user-home


oc new-project capacity-extend-ge

Create a PostgreSQL template that allows storageclass configuration

oc create -f postgresql-persistent-ge.json

oc new-app --name=pg-capacity-extend \
  --template=postgresql-persistent-sc \
  -p STORAGECLASS_NAME=ocs-storagecluster-ceph-rbd \
  -p VOLUME_CAPACITY=30Mi \
  -p POSTGRESQL_USER=student \
  -p POSTGRESQL_PASSWORD=redhat \
  -p POSTGRESQL_DATABASE=capacity-extend-ge \
  -p DATABASE_SERVICE_NAME=pg-capacity-extend-ge


oc status

Verify that the PostgreSQL database did not deploy successfully by checking the pod status and logs.

oc get po

oc logs pod/pg-capacity-extend-ge-1-lgwq5

could not write to file "base/12346/2673": No space left on device

oc get pvc

Notice that the error in previous steps occurred because the PVC size is too small to meet 
PostgreSQL minimum data directory requirements.

oc exec pg-capacity-extend-ge-2-deploy -- df -ah

Extend the size of the PVC.

oc patch pvc/pg-capacity-extend-ge \
  -p '{"spec":{"resources":{"requests": {"storage": "150M"}}}}'


Verify that the PVC is resized successfully.

oc describe pvc pg-capacity-extend-ge

Restart the deployment to finish the resizing process.

oc rollout latest \
  deploymentconfig.apps.openshift.io/pg-capacity-extend-ge

verify

oc get po


Verify the new size of the PVC.

oc get pvc

Verify that the pod mounted the partition with the new PVC size.


oc rsh \
  deploymentconfig.apps.openshift.io/pg-capacity-extend-ge

df -h

/dev/rbd1       139M   47M   92M  34% /var/lib/pgsql/data


Remove the project and the template from the cluster.

oc delete project capacity-extend-ge

oc delete template postgresql-persistent-sc -n openshift


*****
play with postgresql
*****

oc exec -it pg-capacity-extend-ge-2-rw8jp -- psql

oc exec -it pod/pg-capacity-extend-ge-2-rw8jp -c pg-capacity-extend-ge-2-rw8jp -- psql -d capacity-extend-ge -U student


\q

\conninfo
You are connected to database "capacity-extend-ge" as user "student" via socket in "/var/run/postgresql" at port "5432".

listing databases

\list
\l

switching databases

\connect
\c

\c postgres
\c capacity-extend-ge

what is current database?

SELECT current_database();

listing tables

\dt

CREATE TABLE leads (id INTEGER PRIMARY KEY, name VARCHAR);

include size and descr info:

\dt+

SELECT *
FROM pg_catalog.pg_tables
WHERE schemaname != 'pg_catalog' AND 
    schemaname != 'information_schema';

select * from pg_catalog.pg_tables;

create table foo(i int);
insert into foo VALUES(1),(2);



SELECT *
  FROM information_schema.columns
 WHERE table_schema = 'your_schema'
   AND table_name   = 'your_table'
     ;

SELECT *
  FROM information_schema.columns
 WHERE table_name   = 'leads'
     ;

SELECT column_name
  FROM information_schema.columns
 WHERE table_name   = 'leads'
     ;






Error from server: error dialing backend: remote error: tls: internal error

oc get csr -o name | xargs oc adm certificate approve


Ramon

De PVC wordt uiteindelijk een PV, als je het csi.volumeAttribute imageName van de PV er bij pakt - even opzoeken via 

oc describe pv pv_name

dan kan je dat als input gebruiken voor 

rbd disk-usage id -p ocs-storagecluster-cephblockpoo

Het rbd commando zit in dezelfde toolbox als ceph commando. Voorbeeldje:

rbd disk-usage csi-vol-b485d0a0-9e36-11ed-a8f5-0a580a800406 -p ocs-storagecluster-cephblockpool

e.g. pod wks-minio-ibm-minio-kes-pvc

oc describe pv pvc-9a452e6c-e20a-4101-8c35-dde3877fbdf7|grep -i imageName

imageName=csi-vol-ae5c3c0d-7d2a-11ed-a7ab-0a580a81020b



oc exec -ti \
  pod/rook-ceph-operator-6f4568cf4c-hrzvc -n openshift-storage \
  -c rook-ceph-operator -- /bin/bash


rbd disk-usage csi-vol-b20e56a2-7d43-11ed-b27d-0a580a800408 -p ocs-storagecluster-cephblockpool -c /var/lib/rook/openshift-storage/openshift-storage.config


NAME                                          PROVISIONED  USED
csi-vol-ae5c3c0d-7d2a-11ed-a7ab-0a580a81020b       10 GiB  280 MiB


all volumes:

rbd disk-usage -p ocs-storagecluster-cephblockpool -c /var/lib/rook/openshift-storage/openshift-storage.config




****
Very useful:
****

From 4.10 kan de Ceph toolbox enabled worden via:

oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'

Er komt een Pod bij na de patch:

rook-ceph-tools-7c8c77bd96-bb6ds

in openshift-storage project

oc project openshift-storage
Now using project "openshift-storage" on server "https://api.ocp410.tec.uk.ibm.com:6443".

[root@bastion ~]# oc rsh rook-ceph-tools-7c8c77bd96-bb6ds

ceph osd status

ceph osd df

ceph osd utilization

ceph osd pool stats

ceph osd tree

ceph pg stat

ceph osd lspools

ceph health detail

ceph osd pool ls detail

rados df

ceph
ceph> status







