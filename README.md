# VCF_workload

Operations guide to deploy new workload domain  in VMware VCF 
-----------------------

# 1. Introduction

This document discusses the steps involved in deploying new workload domains using VMware's VCF API via Ansible. Though there are multiple ways to deploy new workload domain like utilizing SDDC GUI, Code Stream and using API, it is found that  using Red Hat's Ansible will be more advatageous over other methods due to extensive logging provided when debugging is enabled and less error prone. 

This Workload domain is split into small subtasks as discussed below.

## 1.1. Code 

Download the code from Git.Once downloaded, copy workload_

```bash
$git clone http://location
$git clone --depth=1 --branch=master git://bitbucket.org/sddc/xyz tempDir
$cp roles /etc/ansible/roles


```

## 1.2. Prerequisites

* Ansible 
* Install Git
* Download Dell OpenManage ansible tools with RedFish API from 
``` bash
$git clone -b master --single-branch  https://github.com/dell/dellemc-openmanage-ansible-modules.git
# Install modules into the system by running python 
$python install.py
```



# 2. Building infrastructure

During this phase, input file is needs to be provided with iDRAC informatin ( Assuming Dell Hardware for future deployments).  Ansible will use *redfish api * to reboot the servers.

```bash
# IP address of iDRAC or FQDN of iDRAC is required for this task 
10.10.11.20
10.10.11.21
10.10.11.22
10.10.11.23
```



*Assumptions and requirements* 
* MAC Addresses are available
* DHCP is configured to provide IP address for MAC address with proper maping
* IP helper and PortFast is enabled to forward IP helper requests to DHCP servers
* License keys for vCenter, ESXi node, NSXi licenses are procured before hand
* Network information is ready 
* DNS has valid entries for NSX, vCenter, ESXi nodes along with VIPs
  

# 3. Building WorkLoad domain 

VMware VCF provides ability to interact with SDDC manager using API to build work load domain. This is multi step process. Most of these tasks are modulerized(roles in ansible world) to make it reusable for future enhancements.

VCF API enables us to create network pools, add hosts , add worload domain. These 3 major tasks are seperated to their own ansible roles.


## 3.1. Deploying network pools

Getting API token from SDDC manager is an important step to interact with API .This ansible roles is called by all subsequent tasks we do with SDDC. We need to provide SDDC URL and credentails as input parameters. * Password can be put in ansible vault for security which is encrypted using AES * 



```bash
# Sample input file is provided for reference.This needs to be copied into ansible role deploy_network_esxi_hosts under files section 
$cat network-json.json
{
  "name": "sfo-v01-clst01-np01",
  "networks": [
    {
      "type": "VSAN",
      "vlanId": 1108,
      "mtu": 1500,
      "subnet": "172.17.108.0",
      "mask": "255.255.255.0",
      "gateway": "172.17.108.253",
      "ipPools": [
        {
          "start": "172.17.108.15",
          "end": "172.17.108.20"
        }
      ]
    },
    {
      "type": "VMOTION",
      "vlanId": 1109,
      "mtu": 1500,
      "subnet": "172.17.109.0",
      "mask": "255.255.255.0",
      "gateway": "172.17.109.253",
      "ipPools": [
        {
          "start": "172.17.109.15",
          "end": "172.17.109.20"
        }
      ]
    }
  ]
}

# Once this file is available in roles seciton, then the playbook to deploy network is invoked using the following.   use -vvv option to turn on debugging
$ansible-playbook deploy_networks.yml 

```




## 3.2. Deploying ESXi hosts and getting their UIDs

To deploy ESXi hosts, Network ID information is required. That information is retrieved from steps descripted in section 3.1

```bash
$cat workload_deploy/files/host.json 
[
  {
    "fqdn": "esxi05.sfo01.rainpole.local",
    "username": "root",
    "password": "VMware1!",
    "storageType": "VSAN",
    "networkPoolId": "9457c6bb-b428-450b-b36e-b08bee957348"
  },
  {
    "fqdn": "esxi06.sfo01.rainpole.local",
    "username": "root",
    "password": "VMware1!",
    "storageType": "VSAN",
    "networkPoolId": "9457c6bb-b428-450b-b36e-b08bee957348"
  },
  {
    "fqdn": "esxi07.sfo01.rainpole.local",
    "username": "root",
    "password": "VMware1!",
    "storageType": "VSAN",
    "networkPoolId": "9457c6bb-b428-450b-b36e-b08bee957348"
  },
  {
    "fqdn": "esxi08.sfo01.rainpole.local",
    "username": "root",
    "password": "VMware1!",
    "storageType": "VSAN",
    "networkPoolId": "9457c6bb-b428-450b-b36e-b08bee957348"
  }
]

# Ansible playbook is next run to add hosts into VMware VCF SDDC manager, use -vvv option to turn on debugging
$ansible-playbook /root/sddc_deployhosts.yml
.
.
.
TASK [workload_deploy : Display all hosts] **************************************************************************************************************************************************************
task path: /etc/ansible/roles/workload_deploy/tasks/get_hosts.yml:19
ok: [localhost] => {
    "msg": [
        {
            "id": "4937e2d3-6dc3-4018-bd81-fc30041025c6",
            "server": "esxi06.sfo01.rainpole.local"
        },
        {
            "id": "ff76656c-c12e-4246-9c28-7a212827d14e",
            "server": "esxi07.sfo01.rainpole.local"
        },
        {
            "id": "fbaba42c-fa48-439f-bed3-ee25330504f6",
            "server": "esxi08.sfo01.rainpole.local"
        },
        {
            "id": "99401dfd-eda9-4dfb-9ad7-f67633985a1d",
            "server": "esxi05.sfo01.rainpole.local"
        }
    ]
}

.
.
.



# This playbook pauses multiple times to validate input json file and also validate availability of networks and esxi hosts. Once this step is finished, playbook will proceed to deply the hosts upon verification of the task using the ID generated from the API call 
```
This step generates above output ( truncated ) which provides hostname and their ID generated in VCF environment.Those host ID's and also network ID's along with license information is required tod


## 3.3. Deploying Workload Domain

Once Network and ESXi hosts are deployed i.e commisioned into SDDC, those Nework IDs and Host ID's are used in input json file we create for work load domain.  Sample input is available at the VMware VCF Api documentation referred in reference section.This is final step in work load domian creation . * This takes anywhere between 60 - 120 minutes for 4 node domian * 

```bash
$ansible-playbook /root/sddc_workload.yml 

```



# 4. References

* VCF API at https://code.vmware.com/apis/1002/vmware-cloud-foundation
* Redfish API https://www.dmtf.org/standards/redfish
* Redfish GitHub repo https://github.com/DMTF
* Dell Technologies Ansible Module repositories https://github.com/dell/dellemc-openmanage-ansible-modules
  
