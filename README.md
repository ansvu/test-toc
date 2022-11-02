# How to Replace A Failed OCP Control Plane Node Using Assisted Installer On Premise
## Background/Purpose

A node is a virtual or bare-metal machine in a Kubernetes cluster. Worker nodes host your application containers, grouped as pods.   
The control plane nodes also called Master nodes run services that are required to control the Kubernetes cluster. In OpenShift Container Platform, the Master nodes contain more than just the Kubernetes services for managing the OpenShift Container Platform cluster. Having stable and healthy nodes in a cluster is fundamental to the smooth functioning of your hosted application.  What happens now if one of the Master nodes becomes unavailable after a disk failure or a system crash or the master node is completely not functional anymore? How do you safely replace that Master node? What if that cluster had CEPH storage running on the master and you need to do ceph recovery after replacing the failed master node ? 
The following procedure will address all this.  
  
**Note:** This Procedure is not applicable to SNO.


## Environment Setup

The environment we are using for this blog consists of a set of baremetal servers. Our existing cluster is a compact node cluster with three masters running on OCP 4.10.34. We plan to make master-1 ( 192.168.24.90) unavailable by stopping the kubelet and cordon it, then delete all objects and PODs related to master-1 and finally shutdown that node to simulate the control plane failure. We will then replace it with another baremetal server as shown below.

![table](img/table-1.png)
  
  
  
## Initial Setup:
- **OCP Compact Cluster (3 Masters + 0 worker) + ODF deployed Using On premise Assisted Installer**

```basic
$ oc get no
NAME                                                         STATUS   ROLES           AGE     VERSION
master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   31m     v1.23.5+8471591
master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   31m     v1.23.5+8471591
master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   6m23s   v1.23.5+8471591
```
```basic
$ oc get pod -n openshift-storage
NAME                                                              READY   STATUS      RESTARTS      AGE
cluster-cleanup-job-8153fabeb4849fb52e97dcf6f73cf56a-t9c9d        0/1     Completed   0             8m27s
cluster-cleanup-job-a3726552d6d34a7f4e3275448310eccd-fhdzr        0/1     Completed   0             8m28s
cluster-cleanup-job-be195f61a03e75e44106018524ae256c-chvzf        0/1     Completed   0             8m26s
csi-addons-controller-manager-f6b85759c-dp7sq                     2/2     Running     1 (54m ago)   56m
csi-cephfsplugin-provisioner-8494bbc75b-gsd4m                     6/6     Running     0             115s
csi-cephfsplugin-provisioner-8494bbc75b-vggxn                     6/6     Running     0             115s
csi-cephfsplugin-pxwd9                                            3/3     Running     0             115s
csi-cephfsplugin-wsjbw                                            3/3     Running     0             115s
csi-cephfsplugin-zvqv9                                            3/3     Running     0             115s
csi-rbdplugin-j79h9                                               4/4     Running     0             115s
csi-rbdplugin-jqt68                                               4/4     Running     0             115s
csi-rbdplugin-provisioner-84ff69874-76pst                         7/7     Running     0             115s
csi-rbdplugin-provisioner-84ff69874-jvlm9                         7/7     Running     0             115s
csi-rbdplugin-rtdkr                                               4/4     Running     0             115s
noobaa-operator-585d968475-kx5jh                                  1/1     Running     0             60m
ocs-metrics-exporter-699fbb965f-dnsj8                             1/1     Running     0             60m
ocs-operator-855956bbd-5f4gf                                      1/1     Running     1 (54m ago)   60m
odf-console-5948c474bb-4hd4n                                      1/1     Running     0             60m
odf-operator-controller-manager-5694479898-d64c7                  2/2     Running     1 (54m ago)   60m
rook-ceph-crashcollector-8153fabeb4849fb52e97dcf6f73cf56a-vrhcq   1/1     Running     0             9s
rook-ceph-crashcollector-a3726552d6d34a7f4e3275448310eccd-fb6hx   1/1     Running     0             39s
rook-ceph-crashcollector-be195f61a03e75e44106018524ae256c-kwqx5   1/1     Running     0             8s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-6bfb56868r582   2/2     Running     0             9s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-78fcc9cfgjtq9   2/2     Running     0             8s
rook-ceph-mgr-a-587788d87d-twdl9                                  2/2     Running     0             56s
rook-ceph-mon-a-6898d89964-rnvt4                                  2/2     Running     0             105s
rook-ceph-mon-b-66cdc7f76d-d7cxx                                  2/2     Running     0             81s
rook-ceph-mon-c-69ffb5495c-kcpjc                                  2/2     Running     0             69s
rook-ceph-operator-685cc5b78f-lrhmp                               1/1     Running     0             60m
rook-ceph-osd-0-5df77f584b-82pdt                                  2/2     Running     0             24s
rook-ceph-osd-1-567c96b48c-jztnt                                  2/2     Running     0             24s
rook-ceph-osd-2-7bb8f6f584-ldxzg                                  2/2     Running     0             23s
rook-ceph-osd-prepare-ocs-deviceset-0-data-0d488k-ldhjp           0/1     Completed   0             34s
rook-ceph-osd-prepare-ocs-deviceset-1-data-0xhjxb-t5p2x           0/1     Completed   0             34s
rook-ceph-osd-prepare-ocs-deviceset-2-data-0mncbs-c5pvv           0/1     Completed   0             34s
```
## Prerequisites:

- **Assisted Service either online or on premise available** . procedure to install Podman AI can be found [here](https://github.com/openshift/assisted-service/tree/master/deploy/podman)
- **Existing cluster deployed using On premise Assisted Installer (AI) or Online Assisted Installer**  
  **Note:** For Online Assisted Installer, it needs token, so you may get an API token at [Token](https://cloud.redhat.com/openshift/token)
- **OCP Clusterversion 4.10.3x with 1 failed master**
- **Backup ETCD DB** 
  The official document for etcd backup procedure can be found here [ETCD-Backup](https://docs.openshift.com/container-platform/4.9/backup_and_restore/control_plane_backup_and_restore/backing-up-etcd.html)  
  
  
## Simulating the Control plane failure 
 
 - **Make master-1 Node failed**  
  **Note:** We shutdown master-1 node to simulate the failure  
```shell
$ oc get no
NAME                                                         STATUS                        ROLES           AGE    VERSION
master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready                         master,worker   2d7h   v1.23.5+8471591
master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   NotReady,SchedulingDisabled   master,worker   2d7h   v1.23.5+8471591
master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready                         master,worker   2d6h   v1.23.5+8471591
```

## Part A Control Node replacement 
### Backup Master-1 ETCD, Node, BMH and Machine Config 
- **Backup master-1 Node Config Information**  
```shell
$ oc get nodes master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com -o yaml > master-1-node-info.yaml
```
- **Backup master-1 BMH Config**  
```shell
$ oc -n openshift-machine-api get bmh master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com -o yaml > master-1-bmh-orig.yaml
```
- **Backup master-1 Machine Config**
```shell
$ oc -n openshift-machine-api get machine noknom-aicli-bvlvs-master-x -o yaml > noknom-aicli-bvlvs-master-1-orig.yaml
```

Please see the Official document for details of how to remove an unhealthy etcd member can be found here [Remove-Unhealth-ETCD](https://docs.openshift.com/container-platform/4.8/backup_and_restore/control_plane_backup_and_restore/replacing-unhealthy-etcd-member.html#restore-replace-stopped-etcd-member_replacing-unhealthy-etcd-member)  

### Remove Unhealthy ETCD Master-1 Member
- **Checking ETCD Member Status**
```shell
$ oc -n openshift-etcd rsh etcd-master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com etcdctl member list -w table
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |                            NAME                            |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
| faf792d60794068f | started | master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.91:2380 | https://192.168.24.91:2379 |      false |
| a6dfe8be195765a  | started | master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
| 4778993d473f989e | started | master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.90:2380 | https://192.168.24.90:2379 |      false |
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
```
- **Check ETCD Endpoint Health**
```shell
$ oc -n openshift-etcd rsh etcd-master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com etcdctl endpoint health
{"level":"warn","ts":"2022-10-17T03:44:08.529Z","logger":"client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc000172000/192.168.24.90:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.24.90:2379: connect: no route to host\""}
https://192.168.24.91:2379 is healthy: successfully committed proposal: took = 7.901384ms
https://192.168.24.88:2379 is healthy: successfully committed proposal: took = 7.91493ms
https://192.168.24.90:2379 is unhealthy: failed to commit proposal: context deadline exceeded
Error: unhealthy cluster
```
**Note:** Take note master-1 member-ID for next steps  
- **Remove ETCD master-1 Member-ID**
```shell
$ oc -n openshift-etcd rsh etcd-master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
$ etcdctl member remove 4778993d473f989e
Member 4778993d473f989e removed from cluster 42d89c7cb4fd305f
```
- **Check ETCD Member Status Again**  
  **Note:** Make sure master-1 ETCD member is no longer shown on the status  
```shell
$ etcdctl member list -w table
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |                            NAME                            |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
| faf792d60794068f | started | master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.91:2380 | https://192.168.24.91:2379 |      false |
| a6dfe8be195765a  | started | master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
```
### List The Old Secrets for Unhealthy Master-1
```shell
$ oc get secrets -n openshift-etcd|grep master-1
etcd-peer-master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com              kubernetes.io/tls                     2      73m
etcd-serving-master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com           kubernetes.io/tls                     2      73m
etcd-serving-metrics-master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   kubernetes.io/tls                     2      73m
```
- **Remove the old secrets for the unhealthy etcd member that was removed**  
```shell
$ oc get secrets -n openshift-etcd|grep master-1|awk '{print $1}'|xargs oc -n openshift-etcd delete secrets
secret "etcd-peer-master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com" deleted
secret "etcd-serving-master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com" deleted
secret "etcd-serving-metrics-master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com" deleted
```
### Check ETCD Status
```shell
$ oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd
etcd-master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com                5/5     Running     0          54m
etcd-master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com                2/5     NotReady    0          50m
etcd-master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com                5/5     Running     0          52m
```

### Force the etcd redeployment
```shell
$ oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "single-master-recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
```
### Prepare to Delete master-1 Node
- **Check PODs status on Master-1 Node**
```shell
$ oc get po -A -o wide|grep master-1'
```  
It should be PODs still allocated to master-1, then follow next steps to clean them up.
- **Delete Master-1 Node**
```shell
$ oc delete node master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
```
**Note:** Please check this status again to make sure no more PODs allocated / running on master-1 anymore.    
And also make sure that no more pods on master-1
```shell
$ oc get po -o wide -A|grep master-1|wc -l
0
```
### Delete BMH Config For Master-1
```shell
$ oc delete bmh -n openshift-machine-api master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
baremetalhost.metal3.io "master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com" deleted
```
### Delete Machine Config For Master-1
```shell
$ oc delete machine -n openshift-machine-api noknom-aicli-bvlvs-master-1
machine.machine.openshift.io "noknom-aicli-bvlvs-master-1" deleted
```
### Delete Master-1 Host From AI
  
- **Get InfraID**
```shell
$ export AI_URL='http://192.168.24.80:8090' 
$ INFRA_ID=$(curl -s -X GET $AI_URL/api/assisted-install/v2/infra-envs|jq -r '.[].id')
```
- **Get HostID For Master-1**
```shell
$ curl -s X GET $AI_URL/api/assisted-install/v2/infra-envs/$INFRA_ID/hosts|jq -r '.[] | "host_name: " + .requested_hostname + " host_id: " + .id'|grep master-1
host_name: master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: e0d004d2-a73f-470a-287e-d90ddcc2a4a3
```
- **Delete Master-1 Host from AI**
```shell
$ curl -X DELETE $AI_URL/api/assisted-install/v2/infra-envs/$INFRA_ID/hosts/e0d004d2-a73f-470a-287e-d90ddcc2a4a3
```
- **Check Master-1 Host Again (Should be gone)**
```shell
$ curl -s X GET $AI_URL/api/assisted-install/v2/infra-envs/$INFRA_ID/hosts|jq -r '.[] | "host_name: " + .requested_hostname + " host_id: " + .id'
host_name: master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: ebe515c4-f9c8-528d-9b21-3613a16336e5
host_name: master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: a43ced1e-1456-70d5-2798-77e41d084264
```
### Final Check for Master-1 Host Status After Deletion/Cleanup
```diff
+ oc get machine -n openshift-machine-api
NAME                          PHASE     TYPE   REGION   ZONE   AGE
noknom-aicli-bvlvs-master-0   Running                          2d7h
noknom-aicli-bvlvs-master-2   Running                          2d7h
------------------------------------------------------------------------
+ oc get machine -n openshift-machine-api
NAME                          PHASE     TYPE   REGION   ZONE   AGE
noknom-aicli-bvlvs-master-0   Running                          2d7h
noknom-aicli-bvlvs-master-2   Running                          2d7h
------------------------------------------------------------------------
+ oc get bmh -A
NAMESPACE               NAME                                                         STATE       CONSUMER                      ONLINE   ERROR   AGE
openshift-machine-api   master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   unmanaged   noknom-aicli-bvlvs-master-0   true             2d7h
openshift-machine-api   master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   unmanaged   noknom-aicli-bvlvs-master-2   true             2d7h
------------------------------------------------------------------------
+ oc get nodes
NAME                                                         STATUS   ROLES           AGE    VERSION
master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   2d7h   v1.23.5+8471591
master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   2d7h   v1.23.5+8471591
```
### Example Of Master-x NMState
```yaml
dns-resolver:
  config:
    server:
    - 192.168.24.80
interfaces:
  - name: eno1
    description: SSH interface and Data/Cluster network configuration of master-x
    type: ethernet
    state: up
    ipv4:
      address:
      - ip: 192.168.24.87
        prefix-length: 25
      enabled: true
    mac-address: b8:ce:f6:56:b3:ca
routes:
  config:
  - destination: 0.0.0.0/0
    next-hop-address: 192.168.24.1
    next-hop-interface: eno1
```
### Generate Master-x InfraEnv
```shell
CLUSTER_ID=$(curl -s -X GET "$AI_URL/api/assisted-install/v2/clusters?with_hosts=true" -H "accept: application/json" -H "get_unregistered_clusters: false"| jq -r '.[].id')
jq -n --arg SSH_KEY "$(cat ssh_key.txt)" --arg PULL_SECRET $(cat pull_secret.txt) \
      --arg CLUSTER_ID $CLUSTER_ID \
      --arg MASTER_X_NMSTATE_YAML "$(cat master-x-nmstate.yaml)" \
'{
  "kind": "InfraEnv",
  "name": "infraenv-addhost-master-x",
  "additional_ntp_sources": "192.168.24.80",
  "ssh_authorized_key": $SSH_KEY,
  "pull_secret": $PULL_SECRET,
  "static_network_config": [
    {
      "network_yaml": $MASTER_X_NMSTATE_YAML,
      "mac_interface_map": [
        {
          "mac_address": "b8:ce:f6:56:b3:ca",
          "logical_nic_name": "eno1"
        }
      ]
   }
  ],
  "image_type": "minimal-iso",
  "cluster_id": $CLUSTER_ID,
  "openshift_version": "4.10.34",
  "cpu_architecture": "x86_64",
}' > infraenv-addhost-master-x.json
```  
**Note:** master-x-nmstate.yaml, ssh_key.txt and pull_secret.txt are required to be present before generating this infraenv for master-x. 

### Check Master-x InfraENV Contents
```json
{
  "kind": "InfraEnv",
  "name": "infraenv-addhost-master-x",
  "additional_ntp_sources": "192.168.24.80",
  "ssh_authorized_key": "ssh-rsa xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx root@jumphost1",
  "pull_secret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyy",
  "static_network_config": [
    {
      "network_yaml": "dns-resolver:\n  config:\n    server:\n    - 192.168.24.80\ninterfaces:\n  - name: eno1\n    description: SSH interface and Data/Cluster network configuration of master-x\n    type: ethernet\n    state: up\n    ipv4:\n      address:\n      - ip: 192.168.24.87\n        prefix-length: 25\n      enabled: true\n    mac-address: b8:ce:f6:56:b3:ca\nroutes:\n  config:\n  - destination: 0.0.0.0/0\n    next-hop-address: 192.168.24.1\n    next-hop-interface: eno1",
      "mac_interface_map": [
        {
          "mac_address": "b8:ce:f6:56:b3:ca",
          "logical_nic_name": "eno1"
        }
      ]
    }
  ],
  "image_type": "minimal-iso",
  "cluster_id": "925c7ad1-938b-4033-971c-53079b7f8ba0",
  "openshift_version": "4.10.34",
  "cpu_architecture": "x86_64"
}
```
### Start Update InfraEnv For master-x Using Existed InfraEnv ID
```shell
$ INFRA_ID=$(curl -s -X GET $AI_URL/api/assisted-install/v2/infra-envs|jq -r '.[].id')
$ curl -X PATCH -H "Content-Type: application/json" $AI_URL/api/assisted-install/v2/infra-envs/$INFRA_ID -d @./infraenv-addhost-master-x.json
```
### Generate ISO File For Master-x
```shell
$ curl -s $AI_URL/api/assisted-install/v2/infra-envs/$INFRA_ID/downloads/image-url|jq
{
  "expires_at": "0001-01-01T00:00:00.000Z",
  "url": "http://192.168.24.80:8888/images/cd0867a8-30f1-4369-8e14-97dbe27a0eae?arch=x86_64&type=minimal-iso&version=4.10"
}
```
### Download ISO for Master-x to Local
```shell
$ curl http://192.168.24.80:8888/images/cd0867a8-30f1-4369-8e14-97dbe27a0eae?arch=x86_64\&type=minimal-iso\&version=4.10 > noknom-ai-curl-cplane.iso
```
### Boot ISO to Virtual-DVD/CD
- **Wait For Master-x Node First Boot**
- **Check Master-x Node had discovered From AI GUI**  
  **Note:** The status of this new master-x should be in 'ready' state.  

### Patch Master-x Host-Role to Master
- **Get Master-x Host-ID first**  
```shell
$ curl -s X GET http://192.168.24.80:8090/api/assisted-install//v2/infra-envs/$INFRA_ID/hosts|jq -r '.[] | "host_name: " + .requested_hostname + " host_id: " + .id'
host_name: master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: a2ebbd91-b116-fdad-b642-f815780c3476
```
- **Start Patch Master-x HOST-Role As Master**  
cat master-role.json: 
```json
{
  "kind": "Host",
  "host_role": "master"
}
```
```shell
$ curl -X PATCH -H "Content-Type: application/json" $AI_URL/api/assisted-install/v2/infra-envs/$INFRA_ID/hosts/a2ebbd91-b116-fdad-b642-f815780c3476 -d @./master-role.json|jq
...
...
"requested_hostname": "master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com",
"role": "master",
```

- **Check Master-x Host Status**
```shell
$ curl -s X GET http://192.168.24.80:8090/api/assisted-install//v2/infra-envs/$INFRA_ID/hosts|jq -r '.[] | "host_name: " + .requested_hostname + " host_id: " + .id + "  Host_Role: " + .role + " Status: " + .status'|sort -n
host_name: master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: ebe515c4-f9c8-528d-9b21-3613a16336e5  Host_Role: master Status: installed
host_name: master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: a43ced1e-1456-70d5-2798-77e41d084264  Host_Role: master Status: installed
host_name: master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: a2ebbd91-b116-fdad-b642-f815780c3476  Host_Role: master Status: known
...
...
"requested_hostname": "master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com",
"role": "master",
"stage_started_at": "0001-01-01T00:00:00.000Z",
"stage_updated_at": "0001-01-01T00:00:00.000Z",
"status": "known",
"status_info": "Host is ready to be installed",
```
### Start Master-x Host Installation
```
$ curl -s -X POST $AI_URL/api/assisted-install/v2/infra-envs/$INFRA_ID/hosts/a2ebbd91-b116-fdad-b642-f815780c3476/actions/install
```
### Check Cluster Event/Status From CURL or AI GUI
```shell
$ curl -s X GET $AI_URL/api/assisted-install//v2/infra-envs/$INFRA_ID/hosts|jq -r '.[] | "host_name: " + .requested_hostname + " host_id: " + .id + "  Host_Role: " + .role + " Status: " + .status'|sort -n
host_name: master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: ebe515c4-f9c8-528d-9b21-3613a16336e5  Host_Role: master Status: installed
host_name: master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: a43ced1e-1456-70d5-2798-77e41d084264  Host_Role: master Status: installed
host_name: master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com host_id: a2ebbd91-b116-fdad-b642-f815780c3476  Host_Role: master Status: added-to-existing-cluster
```
**Note:** You can see master-x node has already finished the installation but it remained showing as **'added-to-existing-cluster'**, this is a known issue with current implementation of AI.  
From AI GUI will show as follow:
![Installed-Status](img/installed-status.png "Master-x Installed Status without Green-Check")

### Approve CSR for Master-x Node After 2-3 Reboots
Before master-x can join the existing cluster, two CSRs are Pending to be approved.
```shell
$ oc get csr -A | grep Pending|awk '{print $1}' | xargs oc adm certificate 
certificatesigningrequest.certificates.k8s.io/csr-hvvpj approved
certificatesigningrequest.certificates.k8s.io/csr-d8cpn approved
```

Once the CSR got approved new master will join the cluster 
```shell
$ oc get nodes 
NAME                                                         STATUS   ROLES           AGE    VERSION
master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   3d4h   v1.23.5+8471591
master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   3d3h   v1.23.5+8471591
master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   Ready    master,worker   10h    v1.23.5+8471591
```
### Create BMH For Master-x
cat master-x-bmh.yaml:
```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
  namespace: openshift-machine-api
spec:
  automatedCleaningMode: metadata
  bmc:
    address: "" 
    credentialsName: ""
  bootMACAddress: b8:ce:f6:56:b3:ca
  bootMode: UEFI
  consumerRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: Machine
    name: noknom-aicli-bvlvs-master-x
    namespace: openshift-machine-api
  customDeploy:
    method: install_coreos
  externallyProvisioned: true
  online: true
  userData:
    name: master-user-data-managed
    namespace: openshift-machine-api
 ```
```shell
$ oc apply -f master-x-bmh.yaml
$ oc get bmh -n openshift-machine-api
NAME                                                         STATE       CONSUMER                      ONLINE   ERROR   AGE
master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   unmanaged   noknom-aicli-bvlvs-master-0   true             3d4h
master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   unmanaged   noknom-aicli-bvlvs-master-2   true             3d4h
master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   unmanaged   noknom-aicli-bvlvs-master-x   true             10h
```
 **Note:** Please use master-x BMH as reference and update bootMACAddres and names accordingly  

### Create Machine for Master-x
cat master-x-machine.yaml:
```yaml
apiVersion: machine.openshift.io/v1beta1
kind: Machine
metadata:
  annotations:
    machine.openshift.io/instance-state: externally provisioned
    metal3.io/BareMetalHost: openshift-machine-api/master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
  labels:
    machine.openshift.io/cluster-api-cluster: noknom-aicli-bvlvs
    machine.openshift.io/cluster-api-machine-role: master
    machine.openshift.io/cluster-api-machine-type: master
  name: noknom-aicli-bvlvs-master-x
  namespace: openshift-machine-api
spec:
  metadata: {}
  providerSpec:
    value:
      apiVersion: baremetal.cluster.k8s.io/v1alpha1
      customDeploy:
        method: install_coreos
      hostSelector: {}
      image:
        checksum: ""
        url: ""
      kind: BareMetalMachineProviderSpec
      metadata:
        creationTimestamp: null
      userData:
        name: master-user-data-managed
status:
  addresses:
  - address: ""
    type: InternalIP
  - address: 192.168.24.87
    type: InternalIP
  - address: ""
    type: InternalIP
  - address: ""
    type: InternalIP
  - address: ""
    type: InternalIP
  nodeRef:
    kind: Node
    name: master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
```
**Note:** Similar as in the BMH step, please use master-1 Machine as reference and update address and others accordingly such as address, instance-state and other names belong new master-x.   
```shell
$ oc apply -f master-x-machine.yaml
$ oc get machine -n openshift-machine-api
NAMESPACE               NAME                          PHASE         TYPE   REGION   ZONE   AGE
openshift-machine-api   noknom-aicli-bvlvs-master-0   Running                              2d17h
openshift-machine-api   noknom-aicli-bvlvs-master-2   Running                              2d17h
openshift-machine-api   noknom-aicli-bvlvs-master-x   Provisioned                          23s
```
### Update Master-x Node ProviderID from Machine Config
- **Get providerID from machine of master-x**
```shell
$ oc -n openshift-machine-api get machine noknom-aicli-bvlvs-master-x -o yaml|grep providerID
  providerID: baremetalhost:///openshift-machine-api/master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com/7b5784ea-0339-40dc-9e12-11a0a3fde76d
```
- **Start Update Master-x Node and Add providerID to Spec**
```shell
$ oc edit node master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
spec:
  providerID: baremetalhost:///openshift-machine-api/master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com/7b5784ea-0339-40dc-9e12-11a0a3fde76d

Or using oc patch:
$ oc patch node master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com --type=json -p='[{"op":"add","path":"/spec/providerID","value":"baremetalhost:///openshift-machine-api/master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com/7b5784ea-0339-40dc-9e12-11a0a3fde76d"}]'
```
- **Check Machine for master-x again**
```shell
$ oc get machine -n openshift-machine-api
NAME                          PHASE     TYPE   REGION   ZONE   AGE
noknom-aicli-bvlvs-master-0   Running                          3d4h
noknom-aicli-bvlvs-master-2   Running                          3d4h
noknom-aicli-bvlvs-master-x   Running                          10h
```
### Check ETCD Status for Master-x
```shell
$ oc -n openshift-etcd rsh etcd-master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com etcdctl member list -w table
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |                            NAME                            |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
| faf792d60794068f | started | master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.91:2380 | https://192.168.24.91:2379 |      false |
| a6dfe8be195765a  | started | master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
| 2135bc72cb62f1bc | started | master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.87:2380 | https://192.168.24.87:2379 |      false |
+------------------+---------+------------------------------------------------------------+----------------------------+----------------------------+------------+
```
```shell
$ oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd
etcd-master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          10h
etcd-master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          10h
etcd-master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          10h
```
- **Check ETCD Endpoint Health**  
```shell
$ oc -n openshift-etcd rsh etcd-master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com etcdctl endpoint health
https://192.168.24.88:2379 is healthy: successfully committed proposal: took = 6.904324ms
https://192.168.24.87:2379 is healthy: successfully committed proposal: took = 6.841836ms
https://192.168.24.91:2379 is healthy: successfully committed proposal: took = 7.016554ms
```
###  Check MCP Status
```shell
$ oc get mcp 
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-21495418726ab5760fa0eb32c5de7e3a   True      False      False      3              3                   3                     0                      3d4h
worker   rendered-worker-68065efcf2011d3ce3cf93a3f3c2f29c   True      False      False      0              0                   0                     0                      3d4h
```
### Check OCP Cluster Status
```shell
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.10.34   True        False         False      21h     
baremetal                                  4.10.34   True        False         False      3d4h    
cloud-controller-manager                   4.10.34   True        False         False      3d4h    
cloud-credential                           4.10.34   True        False         False      3d4h    
cluster-autoscaler                         4.10.34   True        False         False      3d4h    
config-operator                            4.10.34   True        False         False      3d4h    
console                                    4.10.34   True        False         False      21h     
csi-snapshot-controller                    4.10.34   True        False         False      3d4h    
dns                                        4.10.34   True        False         False      3d4h    
etcd                                       4.10.34   True        False         False      3d4h    
image-registry                             4.10.34   True        False         False      3d4h    
ingress                                    4.10.34   True        False         False      3d4h    
insights                                   4.10.34   True        False         False      3d4h    
kube-apiserver                             4.10.34   True        False         False      3d4h    
kube-controller-manager                    4.10.34   True        False         False      3d4h    
kube-scheduler                             4.10.34   True        False         False      3d4h    
kube-storage-version-migrator              4.10.34   True        False         False      3d4h    
machine-api                                4.10.34   True        False         False      3d4h    
machine-approver                           4.10.34   True        False         False      3d4h    
machine-config                             4.10.34   True        False         False      21h     
marketplace                                4.10.34   True        False         False      3d4h    
monitoring                                 4.10.34   True        False         False      21h     
network                                    4.10.34   True        False         False      3d4h    
node-tuning                                4.10.34   True        False         False      3d4h    
openshift-apiserver                        4.10.34   True        False         False      21h     
openshift-controller-manager               4.10.34   True        False         False      2d5h    
openshift-samples                          4.10.34   True        False         False      3d4h    
operator-lifecycle-manager                 4.10.34   True        False         False      3d4h    
operator-lifecycle-manager-catalog         4.10.34   True        False         False      3d4h    
operator-lifecycle-manager-packageserver   4.10.34   True        False         False      3d1h    
service-ca                                 4.10.34   True        False         False      3d4h    
storage                                    4.10.34   True        False         False      3d4h    
$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.10.34   True        False         3d4h    Cluster version is 4.10.34
```

## Part B Ceph OSD Recovery 
Master-1 failed , we replaced it with Master-x , we need to remove from Ceph cluster master-1 OSD to replace it with master-x OSD.

### Check OSD From Master-1 Replacement
```shell
$ oc get -n openshift-storage pods -l app=rook-ceph-osd -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP             NODE                                                         NOMINATED NODE   READINESS GATES
rook-ceph-osd-0-5df77f584b-2b4d5   0/2     Pending   0          32m   <none>         <none>                                                       <none>           <none>
rook-ceph-osd-1-567c96b48c-jztnt   2/2     Running   0          52m   10.128.0.98    master-0.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   <none>           <none>
rook-ceph-osd-2-7bb8f6f584-ldxzg   2/2     Running   0          52m   10.129.0.106   master-2.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com   <none>           <none>
```
### Scale down the OSD deployment for the OSD to be replaced.
```shell
$ osd_id_to_remove=0
$ oc scale -n openshift-storage deployment rook-ceph-osd-${osd_id_to_remove} --replicas=0
deployment.apps/rook-ceph-osd-0 scaled
```
### Verify The Rook-Ceph-Osd Pod To Be Terminated
```shell
$ oc get -n openshift-storage pods -l ceph-osd-id=${osd_id_to_remove}
No resources found in openshift-storage namespace.
```
### Force Delete Rook Ceph OSD-0 From Master-1 If Not Deleted
```shell
$ oc delete -n openshift-storage pod rook-ceph-osd-0-5df77f584b-2b4d5 --grace-period=0 --force
```
### Remove Any old OSD Job From the Cluster 
Delete any old ocs-osd-removal jobs. So That a new OSD can be added.
```shell
$ oc delete -n openshift-storage job ocs-osd-removal-job
```
### Remove the old OSD-0 (Master-1) from the cluster
```shell
$ oc process -n openshift-storage ocs-osd-removal -p FAILED_OSD_IDS=${osd_id_to_remove} |oc create -n openshift-storage -f -
job.batch/ocs-osd-removal-job created
```  
**Note:** If face this error 'cephosd: osd.0 is NOT be ok to destroy, retrying in 1m until success' 

**Note:** if job can't be completed you can use the option FORCE_OSD_REMOVAL=true to force the removal like below: 
```shell
$ oc process -n openshift-storage ocs-osd-removal -p FAILED_OSD_IDS=0 -p FORCE_OSD_REMOVAL=true |oc create -n openshift-storage -f -
job.batch/ocs-osd-removal-job created
```
### Verify OCS OSD Removal Job
```shell
$ oc get job -n openshift-storage
NAME                                                   COMPLETIONS   DURATION   AGE
ocs-osd-removal-job                                    0/1           4s         4s
```
### Verify OSD-0(Master-1) Removed successfully
**Note:** Check the status of the ocs-osd-removal pod. A status of Completed confirms that the OSD removal job succeeded.
```shell
$ oc get pod -l job-name=ocs-osd-removal-job -n openshift-storage
NAME                        READY   STATUS      RESTARTS   AGE
ocs-osd-removal-job-dp22s   0/1     Completed   0          73s
```

### Thing Might Goes Wrong For Removal Job
If the removal job is running for a long time, then it is worth it to check the removal jobs log.
```shell
$ oc get pod -l job-name=ocs-osd-removal-job -n openshift-storage
NAME                        READY   STATUS    RESTARTS   AGE
ocs-osd-removal-job-dfqls   1/1     Running   0          34s

$ oc logs -l job-name=ocs-osd-removal-job -n openshift-storage -f
```
### Delete Persistent Volume(PV) From Master-1
- **Find the persistent volume (PV) that need to be deleted by the command**
```shell
$ oc get pv -L kubernetes.io/hostname | grep localblock | grep Released
local-pv-f07262ba    446Gi      RWO     Delete    Released   openshift-storage/ocs-deviceset-0-data-0d488k   localblock-sc   110m   master-1.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com
```
- **Delete the Persistent Volume From Master-1**
```shell
$ oc delete pv local-pv-f07262ba
persistentvolume "local-pv-f07262ba" deleted
```

### Label Master-x Node
```shell
$ oc label node master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com cluster.ocs.openshift.io/openshift-storage=""
```
---
### Checking New Disk Provisioning for Master-x
```diff
+ oc get localvolumeset -n openshift-local-storage
NAMESPACE                 NAME          STORAGECLASS    PROVISIONED   AGE
openshift-local-storage   local-block   localblock-sc   2             113m
--------------------------------------------------------------------------
+ oc describe localvolumeset local-block -n openshift-local-storage
Normal  DiscoveredNewDevice  63s     localvolumeset-symlink-controller  master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com - found possible matching disk, waiting 1m0s to claim
Normal  FoundMatchingDisk    0s      localvolumeset-symlink-controller  master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com - provisioning matching disk
--------------------------------------------------------------------------
+ oc get pod -n openshift-local-storage
NAME                                      READY   STATUS    RESTARTS   AGE
diskmaker-manager-mpmtp                   2/2     Running   0          3m2s

+ oc get -n openshift-storage pods -l app=rook-ceph-osd
NAME                               READY   STATUS    RESTARTS   AGE
rook-ceph-osd-0-97d7fc44d-qnbdm    2/2     Running   0          2m3s
rook-ceph-osd-1-567c96b48c-jztnt   2/2     Running   0          112m
rook-ceph-osd-2-7bb8f6f584-ldxzg   2/2     Running   0          112m
--------------------------------------------------------------------------
+ oc get pv
NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY    STATUS                     CLAIM                          STORAGECLASS                  REASON   AGE
local-pv-424e0b53        446Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-0-data-0hdhnl   localblock-sc                          10m
local-pv-70e23d10        446Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-2-data-0mncbs   localblock-sc                          123m
local-pv-ca6ebf46        446Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-1-data-0xhjxb   localblock-sc                          123m
```
---
### Check Ceph Health Status
```yaml
$ oc rsh -n openshift-storage $TOOLS_POD ceph -s
  cluster:
    id:     353bee71-7698-448e-991e-4037d161f32e
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum b,c,d (age 3m)
    mgr: a(active, since 97m)
    mds: 1/1 daemons up, 1 hot standby
    osd: 3 osds: 3 up (since 2m), 3 in (since 2m)
    rgw: 1 daemon active (1 hosts, 1 zones)
 
  data:
    volumes: 1/1 healthy
    pools:   11 pools, 177 pgs
    objects: 484 objects, 180 MiB
    usage:   318 MiB used, 1.3 TiB / 1.3 TiB avail
    pgs:     177 active+clean
 
  io:
    client:   1.2 KiB/s rd, 4.0 KiB/s wr, 2 op/s rd, 0 op/s wr
 
$ oc rsh -n openshift-storage $TOOLS_POD ceph osd tree
ID   CLASS  WEIGHT   TYPE NAME                                                                STATUS  REWEIGHT  PRI-AFF
 -1         1.30856  root default                                                                                      
 -8         0.43619      rack rack0                                                                                    
 -7         0.43619          host master-0-noknom-aicli-hubcluster-1-lab-eng-cert-redhat-com                           
  1    ssd  0.43619              osd.1                                                            up   1.00000  1.00000
-12         0.43619      rack rack1                                                                                    
-11         0.43619          host master-2-noknom-aicli-hubcluster-1-lab-eng-cert-redhat-com                           
  2    ssd  0.43619              osd.2                                                            up   1.00000  1.00000
 -4         0.43619      rack rack2                                                                                    
 -3         0.43619          host master-x-noknom-aicli-hubcluster-1-lab-eng-cert-redhat-com                           
  0    ssd  0.43619              osd.0                                                            up   1.00000  1.00000
```

### Other Ceph Checking Relate TO Master-x
```shell
$ oc get po -o wide -n openshift-storage|grep master-x 
csi-cephfsplugin-zx8jg                                            3/3     Running     0              29m     192.168.24.87   master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com 
csi-rbdplugin-g62ww                                               4/4     Running     0              29m     192.168.24.87   master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com 
rook-ceph-crashcollector-ad1ab2b253b294f5002578c98c296d33-9pt7h   1/1     Running     0              4m46s   10.130.0.34     master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com 
rook-ceph-mon-d-7cd6c5544b-wczwt                                  2/2     Running     0              5m6s    10.130.0.33     master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com 
rook-ceph-osd-0-97d7fc44d-qnbdm                                   2/2     Running     0              3m41s   10.130.0.37     master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com 
rook-ceph-osd-prepare-ocs-deviceset-0-data-0hdhnl-nzpqn           0/1     Completed   0              8m1s    10.130.0.36     master-x.noknom-aicli.hubcluster-1.lab.eng.cert.redhat.com 

$ oc get pods -n openshift-storage
NAME                                                              READY   STATUS      RESTARTS      AGE
cluster-cleanup-job-a3726552d6d34a7f4e3275448310eccd-fhdzr        0/1     Completed   0             121m
cluster-cleanup-job-be195f61a03e75e44106018524ae256c-chvzf        0/1     Completed   0             121m
csi-addons-controller-manager-f6b85759c-dp7sq                     2/2     Running     2 (99m ago)   169m
csi-cephfsplugin-provisioner-8494bbc75b-46fkc                     6/6     Running     0             93m
csi-cephfsplugin-provisioner-8494bbc75b-gsd4m                     6/6     Running     0             115m
csi-cephfsplugin-pxwd9                                            3/3     Running     0             115m
csi-cephfsplugin-zvqv9                                            3/3     Running     0             115m
csi-cephfsplugin-zx8jg                                            3/3     Running     0             28m
csi-rbdplugin-g62ww                                               4/4     Running     0             28m
csi-rbdplugin-jqt68                                               4/4     Running     0             115m
csi-rbdplugin-provisioner-84ff69874-76pst                         7/7     Running     0             115m
csi-rbdplugin-provisioner-84ff69874-kldq6                         7/7     Running     0             93m
csi-rbdplugin-rtdkr                                               4/4     Running     0             115m
noobaa-core-0                                                     1/1     Running     0             112m
noobaa-db-pg-0                                                    1/1     Running     0             98m
noobaa-endpoint-5745b8b46b-t924q                                  1/1     Running     0             98m
noobaa-operator-585d968475-kx5jh                                  1/1     Running     1 (99m ago)   174m
ocs-metrics-exporter-699fbb965f-dnsj8                             1/1     Running     0             174m
ocs-operator-855956bbd-5f4gf                                      1/1     Running     2 (99m ago)   174m
ocs-osd-removal-job-dp22s                                         0/1     Completed   0             7m47s
odf-console-5948c474bb-4hd4n                                      1/1     Running     0             174m
odf-operator-controller-manager-5694479898-d64c7                  2/2     Running     2 (99m ago)   174m
rook-ceph-crashcollector-a3726552d6d34a7f4e3275448310eccd-b82qv   1/1     Running     0             113m
rook-ceph-crashcollector-ad1ab2b253b294f5002578c98c296d33-9pt7h   1/1     Running     0             4m11s
rook-ceph-crashcollector-be195f61a03e75e44106018524ae256c-kwqx5   1/1     Running     0             113m
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-6bfb56864vb8f   2/2     Running     0             98m
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-78fcc9cfgjtq9   2/2     Running     0             113m
rook-ceph-mgr-a-587788d87d-kj85p                                  2/2     Running     0             98m
rook-ceph-mon-b-66cdc7f76d-d7cxx                                  2/2     Running     0             114m
rook-ceph-mon-c-69ffb5495c-kcpjc                                  2/2     Running     0             114m
rook-ceph-mon-d-7cd6c5544b-wczwt                                  2/2     Running     0             4m31s
rook-ceph-operator-685cc5b78f-lrhmp                               1/1     Running     0             174m
rook-ceph-osd-0-97d7fc44d-qnbdm                                   2/2     Running     0             3m6s
rook-ceph-osd-1-567c96b48c-jztnt                                  2/2     Running     0             113m
rook-ceph-osd-2-7bb8f6f584-ldxzg                                  2/2     Running     0             113m
rook-ceph-osd-prepare-ocs-deviceset-0-data-0hdhnl-nzpqn           0/1     Completed   0             7m26s
rook-ceph-osd-prepare-ocs-deviceset-1-data-0xhjxb-t5p2x           0/1     Completed   0             113m
rook-ceph-osd-prepare-ocs-deviceset-2-data-0mncbs-c5pvv           0/1     Completed   0             113m
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-a-68dd5c9ttrtf   2/2     Running     0             113m
rook-ceph-tools-7659d89cb6-zzksr                                  1/1     Running     0             82m
```
### Check ODF Ceph Storage Cluster on GUI
![ODF-Ceph-Status](img/odf-success-status.png "Check ODF Ceph Cluster Status on GUI")

### Remove The OCS OSD Job After it is Successful Completed
```shell
$ oc delete -n openshift-storage job ocs-osd-removal-job
job.batch "ocs-osd-removal-job" deleted
```
