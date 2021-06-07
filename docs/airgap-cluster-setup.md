# Airgap OpenShift Installation on IBM Cloud VMWare Solutions Shared based on VMWare Cloud Director
## Overview
Deploy OpenShift on IBM Cloud VMWare Solutions based on VMWare Cloud Director.  This toolkit uses Terraform to automate the OpenShift installation process including the Edge Network configuration, Bastion host creation, OpenShift CoreOS bootstrap, loadbalancer, control and worker node creation. Once provisioned, the VMWare Cloud Director environment gives you complete control of all aspects of you OpenShift environment.

This document helps you to configure "airgapped" clusters without Internet access.

Please follow the steps from main document [high level steps to setup the airgap cluster](../README.md#high-level-steps-for-setting-up-the-cluster-as-airgap-install) using this toolkit

**NOTE**: OpenShift 4.6 or later is supported. If you need 4.5 or earlier, see the [VCD Toolkit](https://github.com/vmware-ibm-jil/vcd_toolkit_for_openshift) or the `terraform-openshift4-vmware pre-4.6` [branch](https://github.com/ibm-cloud-architecture/terraform-openshift4-vmware/tree/pre-4.6)


#### Setting up mirror registry

**NOTE**: If you have a mirror registry already setup  with the OCP images mirrored , in some other VCD by your team, then you can skip setting up the mirror registry and directly create the OCP cluster by following the instructions [Create the airgap cluster from bastion server](#create-the-airgap-cluster-from-bastion-server)

You need a mirror registry to mirror the OCP release images so it can be used to create the OCP cluster further. A simple registry setup instructions can be found [here](https://www.redhat.com/sysadmin/simple-container-registry).

In order to prevent an x509 untrusted CA error during the terraform apply step, you must currently copy your mirror certificate to this directory and trust it. I should be able to fix this in the future.  
```
cp <your mirror cert>/etc/pki/ca-trust/source/anchors/
update-ca-trust
trust list | grep -i "<hostname>"
```
The last command will check to see if the update was successful.

#### Create a mirror for OpenShift 4.6 images
You will need to create your own mirror or use an existing mirror to do an airgapped install. Instructions to create a mirror for OpenShift 4.6 can be found [here](https://docs.openshift.com/container-platform/4.6/installing/install_config/installing-restricted-networks-preparations.html#installing-restricted-networks-preparations).

After following the instructions above, you should have a pull secret file on your server.  You will need to know this path as it will be used to update the tfvars file in the next section.

In order for the accept CSR code to work, you will have to:
```  
podman pull quay.io/openshift/origin-cli:latest
podman tag quay.io/openshift/origin-cli:latest <mirror_fqdn>:<mirror_port>/openshift/origin-cli:latest  
podman push <mirror_fqdn>:<mirror_port>/openshift/origin-cli:latest
````

#### Create a mirror for Redhat Openshift catalogs

You will also need to mirror any operators that you will need and place them in the mirror. Instructions can be found [here](https://docs.openshift.com/container-platform/4.6/operators/admin/olm-restricted-networks.html)

**NOTE** Only follow the instructions for mirroring the catalog images and save the `imageContentSourcePolicy.yaml` and `catalogSource.yaml` that gets generated  in the directory `manifests-<index_image_name>-<random_number>` after the `oc adm catalog mirror` command is completed. These two files needs to be shared with the team who would use this shared registry to get the redhat catalogs setup in their airgap cluster once it will be created by them.


#### Setup airgap pre-requisites

##### Add the mirror creds in the pull-secret.json 

Follow thes steps to add your shared registry credentials in the pull-secret.json [Adding shared mirror registry creds in redhat pull-secret.json](https://docs.openshift.com/container-platform/4.4/installing/install_config/installing-restricted-networks-preparations.html#installation-adding-registry-pull-secret_installing-restricted-networks-preparations)

##### Copy registry cert in case of registry setup in different VCD

This is special step and you have to perform it only if you have your mirror registry setup in other VCD than your current VCD where you are trying to create the OCP cluster.

* User have to manually copy  the registry cert file `domain.crt` (cert file for the registry that was generated in the remote VCD bastion server while the registry was setup) from the shared location by your team, to your current hostmachine where you will be creating the cluster (standard location to keep this file: `/opt/registry/certs/domain.crt`). 

* Further mention it in the `terraform.tfvars` as below : 

```
additionalTrustBundle = "/opt/registry/certs/domain.crt"
```

##### Update the terraform.tfvars airgap parameters



Next update your terraform.tfvars file to create the cluster and enable airgap install. We will have to make changes in below sections.

* Update the trust bundle for your mirror registry following steps [Copy registry cert in case of registry setup in different VCD](#copy-registry-cert-in-case-of-registry-setup-in-different-vcd)


* Update the redhat_pull_secret with the file path that was updated from [Add the mirror creds in the pull-secret.json](#add-the-mirror-creds-in-the-pull-secretjson):

```
openshift_pull_secret = "<path to pull secret JSON file created in the previous section>"
```

* Update the `airgapped` object based on the example below:
  *  Update `enabled = true`
  *  Provide the mirror registry details for the shared mirror registry where all your OCP images are mirrored , this will help to install the OCP cluster further.

```
airgapped = {
      enabled = true
      ocp_ver_rel = "4.6.15"
      mirror_ip = "172.16.0.10"
      mirror_fqdn = "bastion.airgapfull.cdastu.com"
      mirror_port = "5000"
      mirror_repository = "ocp4/openshift4"
      }
```

* Update the `initialization_info` object to set `run_cluster_install` to true as shown in the example below:
  * Updated `run_cluster_install = true` , which will create the OCP cluster
  * Update the `public_bastion_ip` and `cluster_public_ip` with the public ips that are available for your VCD

cluster_public_ip       = "161.xxx.x.xxx"

```
 initialization_info     = {
    public_bastion_ip = "161.xxx.xx.xxx"
    bastion_password = "OCP4All"
    internal_bastion_ip = "172.16.0.10"
    terraform_ocp_repo = "https://github.com/ibm-cloud-architecture/terraform-openshift4-vcd"
    rhel_key = "xxxxxxxxxxxxxxxxxxxxxx"
    machine_cidr = "172.16.0.1/24"
    network_name      = "ocpnet"
    static_start_address    = "172.16.0.150"
    static_end_address      = "172.16.0.220"
    run_cluster_install     = true
    }
```

#### Post install cluster configuration

**Disable Telemetry:** You should edit your pull secret and remove the section that refers to `cloud.openshift.com`. This removes Telemetry and Health Reporting. If you don't do this before installation, you will get an error in the insights operator. After installation, go [here](https://docs.openshift.com/container-platform/4.6/support/remote_health_monitoring/opting-out-of-remote-health-reporting.html) for instructions to disable Telemetry Reporting.

Run this command to stop OpenShift from looking for Operators from the Online source.  

`oc patch OperatorHub cluster --type json  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'`

You may receive an Alert stating `Cluster version operator has not retrieved updates in xh xxm 17s. Failure reason RemoteFailed . For more information refer to https://console-openshift-console.apps.<cluster_id>.<base_domain>.com/settings/cluster/` this is normal and can be ignored.

##### Configure mirrored redhat operators catalog

You will also need to mirror any operators that you will need and place them in the mirror. Instructions can be found [here](https://docs.openshift.com/container-platform/4.6/operators/admin/olm-restricted-networks.html)

You will need to follow the instructions carefully in order to setup imagesources for any operators that you want to install.


#### Storage Configuration

If you need to configure a storageclass, there are a few options.  You can follow these instructions for setting up NFS or rookcephfs [here](airgap-storage.md).

