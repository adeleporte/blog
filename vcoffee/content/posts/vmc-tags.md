---
title: "MIGRATE VMS FROM VMC TO ON-PREM WHILE KEEPING NSX SECURITY TAGS"
date: 2020-03-02T17:07:15+01:00
draft: false
featured: true
tags: [
    "hcx",
    "nsx",
    "vmc",
    "python",
    "code"
]
---


### *DISCLAIMER: The views and opinions expressed on this blog are our own and may not reflect the views and opinions of our employer*

![VMC NSX tags](/vmc-tags.png)

In this post, we’re going to see how to script the migration of some workloads from VMC to a datacenter on-prem, leveraging HCX for a live migration.

At the time of this writing, HCX doesn’t copy the NSX tags when doing a VM migration. The following script is going to automate the following actions:

* pull the NSX tags from the source NSX (in VMC)
    * get an authorization token from VMC
    * pull the NSX tags from VMC
* trigger the HCX migration
    * connect to the local HCX manager (on-prem)
    * get local hcx endpoint information
    * get remote hcx endpoint information
    * get vm information
    * get network information
    * start the migration
* push the NSX tags to the destination NSX (on-prem)
    * get the nsx vm id
    * push the tags

Let’s start by defining some variables that our script will need:

```python
#! /usr/bin/python
import requests
import json
 
# Disable Cert Warnings
import warnings
warnings.filterwarnings('ignore', message='Unverified HTTPS request')
 
# Source Variables
nsx_vmc_url = 'https://nsx-****.rp.vmwarevmc.com/vmc/reverse-proxy/api/orgs/****/sddcs/****/sks-nsxt-manager'
refresh_token = '****'
vm_to_migrate = 'Tiny Linux VM'
 
# Destination Variables
nsx_dc_url = 'https://nsx.dc.vcn.lab'
nsx_dc_username = '****'
nsx_dc_password = '****'
 
# HCX Variables
hcx_url = 'https://hcx-mgr.branch.vcn.lab'
hcx_username = '****'
hcx_password = '****'
dest_network = '****-nsx-ls'
```

Now, we need to define several functions to implement all the tasks.

First, get an authorization token from VMC:

```python
## Get the VMC access token
def GetVmcToken():
    resp = requests.post('https://console.cloud.vmware.com/csp/gateway/am/api/auth/api-tokens/authorize?refresh_token={}'.format(refresh_token))
    resp_parsed = json.loads(resp.text)
    access_token = resp_parsed['access_token']
 
    return access_token
```
Then, get the NSX tags from VMC:

```python
# On VMC, Get Tags for the specified VM
def GetVmcVmTags(vm_name):
    headers = {'Authorization': 'Bearer {}'.format(GetVmcToken())}
    # OLD API resp = requests.get('{}/api/v1/fabric/virtual-machines?display_name={}'.format(nsx_vmc_url, vm_name), headers=headers)
    # POLICY API
    #resp = requests.get('{}/policy/api/v1/infra/realized-state/virtual-machines?enforcement_point_path=/infra/deployment-zones/default/enforcement-points/vmc-enforcementpoint'.format(nsx_vmc_url), headers=headers)
    resp = requests.get('{}/api/v1/fabric/virtual-machines?display_name={}'.format(nsx_vmc_url, vm_name), headers=headers)
 
    vmc_vms = json.loads(resp.text)
 
    for vmc_vm in vmc_vms['results']:
      if vmc_vm.__contains__('tags') and vmc_vm['display_name'] == vm_name:
        tags = vmc_vm['tags']
        print('List of VM tags on VMC: {}'.format(tags))
        return tags   
 
    return None
```

Now that we have the functions to authenticate to VMC and pull the NSX tags, let’s connect to HCX manager to implement the migration:

First, connect to HCX Manager:


```python
def ConnectHcx():
    print('Connect to HCX')
    headers = {'Content-Type': 'application/json'}
    payload = {"username": hcx_username, "password": hcx_password}
    resp = requests.post('{}/hybridity/api/sessions'.format(hcx_url), data=json.dumps(payload), headers=headers, verify=False)
 
    token = resp.headers['x-hm-authorization']
     
 
    return token
```

Then, get the cloud configuration:

```python
def GetCloudConfig(token):
    print('Get Cloud Config')
    headers = {'Content-Type': 'application/json', 'Accept': 'application/json', 'x-hm-authorization': token}
    resp = requests.get('{}/hybridity/api/cloudConfigs'.format(hcx_url), headers=headers, verify=False)
 
    resp_parsed = json.loads(resp.text)
    cloud_config = resp_parsed['data']['items'][1]
 
    return cloud_config
```
Then, get the local endpoint configuration:

```python
def GetLocalHcxEndpoint(token):
    print('Get Local Endpoint')
    headers = {'Content-Type': 'application/json', 'Accept': 'application/json', 'x-hm-authorization': token}
    payload = {'filter': {'cloud': {'remote': False}}}
    resp = requests.post('{}/hybridity/api/service/inventory/resourcecontainer/list'.format(hcx_url), data=json.dumps(payload), headers=headers, verify=False)
 
    resp_parsed = json.loads(resp.text)
    hcx_endpoint = resp_parsed['data']['items'][0]
 
    source = {'computeResourceId': hcx_endpoint['vcuuid'],
              'endpointId': hcx_endpoint['endpoint']['endpointId'],
              'endpointName': hcx_endpoint['endpoint']['name'],
              'endpointType': hcx_endpoint['endpoint']['cloudType'],
              'resourceId': hcx_endpoint['resourceId'],
              'resourceName': hcx_endpoint['name'],
              'resourceType': hcx_endpoint['resourceType'],
              }
 
    return source
```

And the remote endpoint configuration:

```python
def GetRemoteHcxEndpoint(token):
    print('Get Remote Endpoint')
    headers = {'Content-Type': 'application/json', 'Accept': 'application/json', 'x-hm-authorization': token}
    payload = {'filter': {'cloud': {'remote': True}}}
    resp = requests.post('{}/hybridity/api/service/inventory/resourcecontainer/list'.format(hcx_url), data=json.dumps(payload), headers=headers, verify=False)
 
    resp_parsed = json.loads(resp.text)
    hcx_endpoint = resp_parsed['data']['items'][1]
 
    destination = {'computeResourceId': hcx_endpoint['vcuuid'],
              'endpointId': hcx_endpoint['endpoint']['endpointId'],
              'endpointName': hcx_endpoint['endpoint']['name'],
              'endpointType': hcx_endpoint['endpoint']['cloudType'],
              'resourceId': hcx_endpoint['resourceId'],
              'resourceName': hcx_endpoint['name'],
              'resourceType': hcx_endpoint['resourceType'],
              }
 
    return destination
```

Now, we need to get the information about the VM to migrate:

```python
def GetHcxVm(token, vm_name):
    print('Get Hcx Vm')
    headers = {'Content-Type': 'application/json', 'Accept': 'application/json', 'x-hm-authorization': token}
    payload = {'filter': {'cloud': {'remote': True}}}
    resp = requests.post('{}/hybridity/api/service/inventory/virtualmachines'.format(hcx_url), data=json.dumps(payload), headers=headers, verify=False)
 
    resp_parsed = json.loads(resp.text)
    hcx_vms = resp_parsed['data']['items']
 
    for hcx_vm in hcx_vms:
        if hcx_vm['name'] == vm_name:
            #print(hcx_vm)
            entityDetails = {'entityId': hcx_vm['id'], 'entityName': hcx_vm['name']}
            return entityDetails
 
    print('\033[91mFail to find the VM...\033[0m')
    exit(1)
    return None
```

and the network used by the VM to migrate:

```python
def GetHcxNetworks(token, vm_name):
    print('Get Hcx Network')
    headers = {'Content-Type': 'application/json', 'Accept': 'application/json', 'x-hm-authorization': token}
    payload = {'filter': {'cloud': {'remote': True}}}
    resp = requests.post('{}/hybridity/api/service/inventory/virtualmachines'.format(hcx_url), data=json.dumps(payload), headers=headers, verify=False)
 
    resp_parsed = json.loads(resp.text)
    hcx_vms = resp_parsed['data']['items']
 
    for hcx_vm in hcx_vms:
        if hcx_vm['name'] == vm_name:
            targetNetworks = [{
                            'destNetworkDisplayName': '{}'.format(dest_network),
                            'destNetworkHref': '/infra/segments/{}'.format(dest_network),
                            'destNetworkName': '{}'.format(dest_network),
                            'destNetworkType': "NsxtSegment",
                            'destNetworkValue': '/infra/segments/{}'.format(dest_network),
                            'srcNetworkDisplayName': hcx_vm['network'][0]['displayName'],
                            'srcNetworkHref': hcx_vm['network'][0]['value'],
                            'srcNetworkName': hcx_vm['network'][0]['name'],
                            'srcNetworkType': hcx_vm['network'][0]['type'],
                            'srcNetworkValue': hcx_vm['network'][0]['value']
                            }]
 
             
            return targetNetworks
 
    print('\033[91mFail to find the network...\033[0m')
    exit(1)
    return None
```

We need to define the VM placement for the migration (basically where the VM is going to be migrated to)

```python
def GetHcxPlacement():
    hcx_placement = [{
                'containerId': "host-16",
                'containerName': "esx-03.branch.vcn.lab",
                'containerType': "host"
                },
                {
                'containerId': "datacenter-2",
                'containerName': "Branch",
                'containerType': "dataCenter"
                }]
    return hcx_placement
```

And the storage to use at the destination:

```python
def GetHcxStorage():
    hcx_storage = {
                'datastoreId': "datastore-61",
                'datastoreName': "Datastore",
                'diskProvisionType': "sameAsSource",
                'storageProfile': {
                'type': "default"
                }}
    return hcx_storage
```

Let’s get the VM id from NSX (at the destination)

```python
# Get the new VM Id
def GetDcVmId():
    headers = {'Content-Type': 'application/json'}
    resp = requests.get('{}/api/v1/fabric/virtual-machines?display_name={}'.format(nsx_dc_url, vm_to_migrate), auth=(nsx_dc_username, nsx_dc_password), headers=headers, verify=False)
    resp_parsed = json.loads(resp.text)
    external_id = resp_parsed['results'][0]['external_id']
    print('DC VM Id:'+external_id)
 
    return external_id
```

And use this Id to push the tags:

```python
# Tag a VM in the DC
def TagDcVm(tags):
    headers = {'Content-Type': 'application/json'}
    payload = {'external_id': GetDcVmId(), 'tags': tags}
     
    print('Tag VM in NSX DC')
    requests.post('{}/api/v1/fabric/virtual-machines?action=update_tags'.format(nsx_dc_url), data=json.dumps(payload), auth=(nsx_dc_username, nsx_dc_password), headers=headers, verify=False)
 
    return None
```

Finally, a function to start the migration is needed:

```python
def HcxMigrate(token, migrations):
    headers = {'Content-Type': 'application/json', 'Accept': 'application/json', 'x-hm-authorization': token}
    payload = migrations
    resp = requests.post('{}/hybridity/api/migrations?action=start'.format(hcx_url), data=json.dumps(payload), headers=headers, verify=False)
 
    resp_parsed = json.loads(resp.text)
 
    id = resp_parsed['migrations'][0]['migrationId']
 
    print('Start migration...')
    return id
```

And another function to follow the progression of the migration:

```python
def GetHcxMigrationStatus(token, migration):
    headers = {'Content-Type': 'application/json', 'Accept': 'application/json', 'x-hm-authorization': token}
    payload = {'filter': {'migrationId': [migration]}}
    resp = requests.post('{}/hybridity/api/migrations?action=query'.format(hcx_url), data=json.dumps(payload), headers=headers, verify=False)
 
    resp_parsed = json.loads(resp.text)
    state = resp_parsed['items'][0]['migrationInfo']['progressDetails']['state']
 
    print('Migration State: '+state)
 
    return state
```

Now that’s we’ve defined all our functions, let’s implement the full logic to implement the migration:

```python
def main():
    print('\033[92mStart migration process...\033[0m')
    # Get tags from the VMC vm
    tags = GetVmcVmTags(vm_to_migrate)
 
    # Migrate the VM
    token = ConnectHcx()
    #print(token)
    GetCloudConfig(token)
    hcx_local_endpoint = GetLocalHcxEndpoint(token)
    hcx_remote_endpoint = GetRemoteHcxEndpoint(token)
    hcx_vms = GetHcxVm(token, vm_to_migrate)
    hcx_networks = GetHcxNetworks(token, vm_to_migrate)
 
    hcx_placement = GetHcxPlacement()
    hcx_storage = GetHcxStorage()
 
    input = {
                'decisionRules': {
                    'forcePowerOffVm': False,
                    'removeISOs': True,
                    'removeSnapshots': True,
                    'upgradeHardware': False,
                    'upgradeVMTools': False
                },
                'destination': hcx_local_endpoint,
                'entityDetails': hcx_vms,
                'migrationType': 'vMotion', # or VR
                'networks': {
                    'retainMac': False,
                    'targetNetworks': hcx_networks
                },
                'placement': hcx_placement,
                'source': hcx_remote_endpoint,
                'schedule': {},
                'storage': hcx_storage
            }
 
    migrations = {'migrations': [{'input': input}]}
    migration_id = HcxMigrate(token, migrations)
 
    while True:
        status = GetHcxMigrationStatus(token, migration_id)
        if (status == 'COMPLETED'):
            print('\033[92mMigration completed...\033[0m')
            break
        if (status == 'FAILED'):
            print('\033[91mMigration failed...\033[0m')
            break
        time.sleep(15)
 
    # Retag the VM in the DC
    TagDcVm(tags)
 
    print('\033[95mMigration process done.\033[0m')
     
  
 
main()
```

Now, let’s test our script. Here is the VM at source, before the migration:
![VM at source, before the migration](/hcx-tags-vm-source.png)

Some tags are applied to the VM
![tags at source, before the migration (scope = scope1, tag = tag1)](/hcx-tags-vm-tags.png)

Start our script:

```bash
adeleporte@adeleporte-a01 hcx-tags % ./main.py
Start migration process...
List of VM tags on VMC: [{u'scope': u'scope1', u'tag': u'tag1'}]
Connect to HCX
Get Cloud Config
Get Local Endpoint
Get Remote Endpoint
Get Hcx Vm
Get Hcx Network
Start migration...
Migration State: PROCESS_COLLECT_GATEWAY_DETAILS
Migration State: PROCESS_COLLECT_GATEWAY_DETAILS
Migration State: PROCESS_COLLECT_GATEWAY_DETAILS
Migration State: PROCESS_COLLECT_VM_DETAILS_RESULT
Migration State: PROCESS_COLLECT_VM_DETAILS_RESULT
Migration State: JOIN_RECONFIGURE
Migration State: PROCESS_RECONFIGURE_RESULT
Migration State: DESTINATION_SIDE_START_RELOCATE
Migration State: PROCESS_START_RELOCATE_RESULT
Migration State: PROCESS_DESTINATION_SIDE_RELOCATE_PROGRESS_RESULT
Migration State: PROCESS_SOURCE_SIDE_RELOCATE_PROGRESS_RESULT
Migration State: SOURCE_SIDE_CLEANUP
Migration State: DESTINATION_SIDE_CLEANUP
Migration State: COMPLETED
Migration completed...
DC VM Id:500c9196-a9db-34c9-237d-afb214161169
Tag VM in NSX DC
Migration process done.
```

The migration is started and the HCX console can be used to follow its progress.

![HCX Migration](/hcx-tags-migration1.png)

When the migration is finished, let’s connect on the DC NSX to check the tags

![HCX Migration](/hcx-tags-migration2.png)
![Tags after migration](/hcx-tags-migration3.png)
