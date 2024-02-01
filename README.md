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
# VM A
oc exec virt-launcher-rhel9-dual-nic-a-vwx6j -- ip a add 192.168.1.10/24 dev eth1
# VM B
oc exec virt-launcher-rhel9-dual-nic-b-dsk1a -- ip a add 192.168.1.11/24 dev eth1
```

### Connectivity test