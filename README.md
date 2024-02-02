# aws-ocp-kubevirt-fsx

## Introduction

Welcome to the technical documentation for setting up a tech stack designed to enable Live Migration feature in KubeVirt, utilizing AWS services and OpenShift. This infrastructure is engineered to provide a reliable, scalable, and secure environment to support containerized applications and take advantage of the ReadWriteMany file system functionality offered by Amazon FSx.


The following technologies make up this tech stack:

- **AWS Baremetal EC2 Instances**: Provided as physically dedicated AWS compute capacity for running KubeVirt, as one requirement for using KubeVirt is to deploy it on metal instances.

- **AWS FSx**: A fully managed file system service for use with on-premises applications or extending the storage capacity of applications running in AWS, specifically used to support ReadWriteMany access required by Live Migration.

- **OpenShift**: An open-source container application platform based on Kubernetes that automatically deploys, manages, and scales applications.

- **KubeVirt**: An open-source extension to Kubernetes for managing virtual machines as native Kubernetes resources.


Throughout the following sections, we will provide an overview of each technology in our stack and instructions on how to deploy these resources using AWS CloudFormation templates and a openshift installation script. It is important to note that this documentation assumes a solid understanding of AWS services, containerization technologies such as Kubernetes and OpenShift, and the prerequisites for enabling Live Migration with KubeVirt.


# Prerequisites
## Openshift-Install
```bash
curl -fsSLO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.4/openshift-install-mac-arm64-4.14.4.tar.gz
tar xzf openshift-install-mac-arm64-4.14.4.tar.gz
./openshift-install --help
```

## OCP Installation Preparation
Create the manifest directory:
```bash
mkdir -p ocp-manifests-dir/
```

Save `install-config.yaml` inside `ocp-manifests-dir`:
```yaml
additionalTrustBundlePolicy: Proxyonly
apiVersion: v1
baseDomain: kloia.rocks
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: 
    aws:
      type: c5.metal
  replicas: 2
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: ocp-demo
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: eu-west-1
publish: External
pullSecret: '...'
sshKey: |

```

Generate the manifests:
```bash
./openshift-install create manifests --dir ocp-manifests-dir
```

# Installation

## Install OCP cluster
```bash
# Backup the `install-config.yaml` and all manifests:
cp -r ocp-manifests-dir/ ocp-manifests-dir-bkp
./openshift-install create cluster --dir ocp-manifests-dir --log-level debug
```

## Provision FSx for ONTAP

We're going to make a multi-Availability Zone file system called FSx for ONTAP in the same private network as the OCP cluster.

Just get the ID for the network (VPC), two IDs for the specific parts of the network where you want the file system, and all the IDs for the paths data takes through the network. Put all these values into a command.

The FSxAllowedCIDR thing is about what kind of internet traffic is allowed to reach the FSx for ONTAP file system. You can let all traffic in by using 0.0.0.0/0, or set specific rules. Run a command in the terminal to make the FSx for ONTAP file system.

Note: If you want the file system to have different space and speed, you can change the default values by adjusting the StorageCapacity and ThroughputCapacity in the CFN template.

```bash
cd fsx

aws cloudformation create-stack \
  --stack-name FSXONTAP \
  --template-body file://./netapp-cf-template.yaml \
  --region <region-name> \
  --parameters \
  ParameterKey=Subnet1ID,ParameterValue=[subnet1_ID] \
  ParameterKey=Subnet2ID,ParameterValue=[subnet2_ID] \
  ParameterKey=myVpc,ParameterValue=[VPC_ID] \
  ParameterKey=FSxONTAPRouteTable,ParameterValue=[routetable1_ID,routetable2_ID] \
  ParameterKey=FileSystemName,ParameterValue=myFSxONTAP \
  ParameterKey=ThroughputCapacity,ParameterValue=256 \
  ParameterKey=FSxAllowedCIDR,ParameterValue=[your_allowed_CIDR] \
  ParameterKey=FsxAdminPassword,ParameterValue=[Define password] \
  ParameterKey=SvmAdminPassword,ParameterValue=[Define password] \
  --capabilities CAPABILITY_NAMED_IAM
```

## Deploy Trident CSI driver

Set Kubeconfig File
```bash
set -gx KUBECONFIG /Users/bilal/Projects/demo/ocp-manifests-dir/auth/kubeconfig
kubectl get nodes
```
Download & Deploy Trident Helm Chart
```bash
oc create ns trident
curl -L -o trident-installer-22.10.0.tar.gz https://github.com/NetApp/trident/releases/download/v22.10.0/trident-installer-22.10.0.tar.gz
tar -xvf ./trident-installer-21.10.1.tar.gz
cd trident-installer/helm 
helm install trident -n trident trident-operator-22.10.0.tgz
```

Create `svm_secret.yaml` file and deploy it for backend to consume
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-fsx-ontap-nas-secret
  namespace: trident
type: Opaque
stringData:
  username: vsadmin
  password: use the password you defined in cloudformation template
```
```bash
oc apply -f svm_secret.yaml
oc get secrets -n trident | grep backend-fsx-ontap-nas
```

Deploy Trident CSI backend

Set up the Trident CSI backend for FSx for ONTAP by configuring how Trident talks to the storage system (FSx for ONTAP). We'll be using the ontap-nas driver to create storage volumes.

Begin by navigating to the `fsx` directory in your git repository clone. Open the `backend-ontap-nas.yaml` file. Replace the `managementLIF` and `dataLIF` in that file with the Management DNS name and NFS DNS name of your Amazon FSx Storage Virtual Machine. Also, replace `svm` with the SVM name.

Please note that you can find `ManagementLIF` and `DataLIF` in the Amazon FSx Console under "Storage virtual machines" Or you can get these information using AWS CLI.

```bash
aws fsx describe-storage-virtual-machines | jq '.StorageVirtualMachines[0].Endpoints | .Management, .Nfs'
# Edit `backend-ontap-nas.yaml`
oc apply -f fsx/backend-ontap-nas.yaml
```

Verify the backend configuration. You should see the that backend status is 'SUCCESS'
```bash
oc get tbc -n trident # verify the configuration
```
Create Storage Class by applying `storage-class-csi-nas.yaml` file
```bash
oc apply -f fsx/storage-class-csi-nas.yaml
```
Verify the status of the trident-csi storage class creation. You should see a storage class named `trident-csi`
```bash
oc get sc
```

## Deploy Kubevirt

```bash
echo '
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
    - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  startingCSV: kubevirt-hyperconverged-operator.v4.14.0
  channel: "stable"' | k apply -f-

# Wait a few minutes until all Pods within the NS "openshift-cnv" are ready.

echo '
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:' | k apply -f-
```

Verification:
```bash
watch oc get csv -n openshift-cnv
oc get kubevirt -n openshift-cnv
oc get HyperConverged -n openshift-cnv
```

# Live Migration of VMs with Kubevirt in OpenShift Cluster

Lets say you have deployed two VMs using Kubevirt in your OpenShift Cluster. Assume that both VMs are running on separate bare-metal worker nodes. Now, due to maintenance reasons, you need to shut down WorkerA node. However, before doing so, you must seamlessly migrate the VM running on WorkerA to the other bare-metal node, WorkerB. Live migration is the preferred method for this task.

One challenge is that when moving Kubevirt VMs to a different machine, the IP address will change. Since Kubevirt VMs are essentially pods within Kubernetes, moving them to another node will result in a change of IP address.

To address this issue, you can add a second network interface to the Kubevirt VMs. This additional interface will allow you to maintain connectivity and avoid disruptions during the live migration process. By doing so, the VMs will have a stable IP address on the new worker node, WorkerB, ensuring a smooth transition.

### Create NAD(NetworkAttachmentDefinition)
```bash
oc create ns vm-test
oc apply -f virtualization/nad.yaml
```

### Create two VMs with two NICS
```bash
oc apply -f virtualization/vm-rhel-9-dual-nic.yaml
```
After the VM is started, check the events:
```bash
$ oc get vm
NAME                  AGE   STATUS    READY
rhel9-dual-nic-a      98m   Running   True
rhel9-dual-nic-b      36s   Running   True
```
This command will create two VMs with two interfaces but since we don't have a DHCP server deployed on our cluster we need to give an IP address to secondary network interface manually.

### Assign IP address for the secondary network interface

```bash
$ oc get vm -n vm-test
NAME               AGE    STATUS    READY
rhel9-dual-nic-a   119m   Running   True
rhel9-dual-nic-b   119m   Running   True
# Assign IP for VM A

$ virtctl console -n vm-test rhel9-dual-nic-a
Successfully connected to rhel9-dual-nic-a console. The escape sequence is ^]

rhel9-dual-nic-a login: cloud-user
Password:
Last login: Thu Feb  1 12:24:15 on tty1
[cloud-user@rhel9-dual-nic-a ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8901 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:12 brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 10.0.2.2/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 86306478sec preferred_lft 86306478sec
    inet6 fe80::a6:a2ff:fe00:12/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1300 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:13 brd ff:ff:ff:ff:ff:ff
    altname enp2s0

[cloud-user@rhel9-dual-nic-a ~]$ sudo ip a add 192.168.1.10/24 dev eth1
[cloud-user@rhel9-dual-nic-a ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8901 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:12 brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 10.0.2.2/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 86306385sec preferred_lft 86306385sec
    inet6 fe80::a6:a2ff:fe00:12/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1300 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:13 brd ff:ff:ff:ff:ff:ff
    altname enp2s0
    inet 192.168.1.10/24 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::16b6:cf73:eb06:e689/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

# VM B
$ virtctl console -n vm-test rhel9-dual-nic-b
Successfully connected to rhel9-dual-nic-b console. The escape sequence is ^]

rhel9-dual-nic-b login: cloud-user
Password:
[cloud-user@rhel9-dual-nic-b ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8901 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:14 brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 10.0.2.2/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 86306229sec preferred_lft 86306229sec
    inet6 fe80::a6:a2ff:fe00:14/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1300 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:15 brd ff:ff:ff:ff:ff:ff
    altname enp2s0
    inet6 fe80::5d9a:f417:8506:4554/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
[cloud-user@rhel9-dual-nic-b ~]$ sudo ip a add 192.168.1.11/24 dev eth1
[cloud-user@rhel9-dual-nic-b ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8901 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:14 brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 10.0.2.2/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 86306205sec preferred_lft 86306205sec
    inet6 fe80::a6:a2ff:fe00:14/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1300 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:a6:a2:00:00:15 brd ff:ff:ff:ff:ff:ff
    altname enp2s0
    inet 192.168.1.11/24 scope global eth1
       valid_lft forever preferred_lft forever
```

After these steps you can see that we have an IPv4 address on the eth1 interface in both VMs. Normally we could do this automatically with the help of a DHCP server, but in this example we have manually assigned IP addresses. Now we will test the communication between these VMs.

### Connectivity test

```bash
[cloud-user@rhel9-dual-nic-a ~]$ ip a | grep '192'
    inet 192.168.1.10/24 scope global eth1
[cloud-user@rhel9-dual-nic-a ~]$ ping 192.168.1.11
PING 192.168.1.11 (192.168.1.11) 56(84) bytes of data.
64 bytes from 192.168.1.11: icmp_seq=1 ttl=64 time=0.873 ms
64 bytes from 192.168.1.11: icmp_seq=2 ttl=64 time=0.405 ms
64 bytes from 192.168.1.11: icmp_seq=3 ttl=64 time=0.524 ms
64 bytes from 192.168.1.11: icmp_seq=4 ttl=64 time=0.207 ms
64 bytes from 192.168.1.11: icmp_seq=5 ttl=64 time=0.487 ms
64 bytes from 192.168.1.11: icmp_seq=6 ttl=64 time=0.387 ms

--- 192.168.1.11 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 5095ms
rtt min/avg/max/mdev = 0.207/0.480/0.873/0.202 ms

[cloud-user@rhel9-dual-nic-b ~]$ ip a | grep '192'
    inet 192.168.1.11/24 scope global eth1
[cloud-user@rhel9-dual-nic-b ~]$ ping 192.168.1.10
PING 192.168.1.10 (192.168.1.10) 56(84) bytes of data.
64 bytes from 192.168.1.10: icmp_seq=1 ttl=64 time=1.47 ms
64 bytes from 192.168.1.10: icmp_seq=2 ttl=64 time=0.561 ms
64 bytes from 192.168.1.10: icmp_seq=3 ttl=64 time=0.315 ms
64 bytes from 192.168.1.10: icmp_seq=4 ttl=64 time=0.443 ms
64 bytes from 192.168.1.10: icmp_seq=5 ttl=64 time=0.486 ms
64 bytes from 192.168.1.10: icmp_seq=6 ttl=64 time=0.549 ms

--- 192.168.1.10 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 5042ms
rtt min/avg/max/mdev = 0.315/0.636/1.465/0.379 ms
```

As you can see from the output, each VM can communicate over the secondary network interface. And that means if VM A is evicted with LiveMigration strategy, the IP address on the secondary network interfaces will still be same and the VM B can communicate the VM A while its migrating to other node.