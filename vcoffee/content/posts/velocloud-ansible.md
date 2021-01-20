---
title: "How to write an Ansible module for Velocloud"
date: 2020-02-27T17:07:15+01:00
draft: false
---


### *DISCLAIMER: The views and opinions expressed on this blog are our own and may not reflect the views and opinions of our employer*

In this post, we’re going to cover the different steps involved to create a basic Ansible module for the Velocloud SDWAN Solution.

![VCO](/vco.png)

As an example, we’ll start with a quite basic module: how to configure the static routing at the Edge Level.

VMware provides a very good API publicly available at http://code.vmware.com

Let’s start with the basic structure of an Ansible module. As you may know, modules are generally written in Python. Basically, we need to import the Ansible python libraries and use the AnsibleModule function as the entry point of our module.

This module uses a dictionary structure to define the arguments list of the Ansible module. In our example, we’ll need some information about the Velocloud Orchestrator to connect to, the tenant and edge that we want to configure, and some information about the static route (subnet, prefix, nexthop, interface…). Finally, a state argument is used by the module to specify if the route should be created or deleted.

Here is the basic module with these arguments:

```python
from ansible.module_utils.basic import AnsibleModule
import urllib3
 
def main():
    urllib3.disable_warnings()
    module = AnsibleModule(
        argument_spec = dict(
            host                = dict(required=True),
            username            = dict(required=True),
            password            = dict(required=True),
            operator            = dict(required=True, type='bool'),
            enterprise          = dict(required=True),
            edge                = dict(required=True),
            state               = dict(default='present', choices=['present', 'absent']),
            subnet              = dict(required=True),
            prefix              = dict(required=True),
            nexthop             = dict(required=True),
            interface           = dict(required=True),
            cost                = dict(default=0, type='int'),
            advertise           = dict(default=True, type='bool'),
            preferred           = dict(default=True, type='bool'),
            description         = dict(required=False)
        )
    )
 
if __name__ == '__main__':
    main()
```

Now that we have our Ansible module, let’s code the Velocloud logic to be able to configure some static routes on a Edge, we need to go through these steps:

    get the tenant ID
    get the edge ID within this tenant
    get the “DeviceSettings” configuration module for this edge
    modify this module to create/modify/delete the routes

First, we need a client to be able to connect to the Velocloud SDWAN API. We use this client: https://code.vmware.com/samples/5554/velocloud-orchestrator-json-rpc-api-client—python

This client is straightforward to use. We just have to create the client by calling the VcoRequestManager class and use the “authenticate” method with correct credentials

Here is the code we use, just after the AnsibleModule function initialization:

```python
try:
        client = VcoRequestManager(hostname=module.params['host'], verify_ssl=False)
        client.authenticate(module.params['username'], module.params['password'], is_operator=module.params['operator'])
    except ApiException as e:
        module.fail_json(msg=e)
```
Once authenticated, we need to retrieve the enterprise Id. Let’s define a function to get this Id:

```python
def get_enteprise(client, module):
    params = { "networkId": 1, "with": [] }
    enterprises = client.call_api('/network/getNetworkEnterprises', params)
 
    for enterprise in enterprises:
            if enterprise['name'] == module.params['enterprise']:
                return enterprise
                break
    module.fail_json(msg='Cant find enterprise')
```

Then, we need to get the Edge Id:

```python
def get_edge(client, module, ent_id):
    params = { "enterpriseId": ent_id, "with": [] }
    edges = client.call_api('/enterprise/getEnterpriseEdges', params)
 
    for edge in edges:
            if edge['name'] == module.params['edge']:
                return edge
                break
    module.fail_json(msg='Cant find edge') 
```

Then, the “deviceSettings” module ID for this Edge:

```python
def get_device_module(client, module, ent_id, edge_id):
    params = { 'enterpriseId': ent_id, 'edgeId': edge_id }
    modules = client.call_api('/edge/getEdgeConfigurationStack', params)
 
    for module in modules[0]['modules']:
            if module['name'] == 'deviceSettings':
                return module
                break
    module.fail_json(msg='Cant find device settings module')
```

And finally, we can modify this module to create/update/delete the routes:

```python
def add_static_route(client, module, ent_id, dev):
    route = {
        "destination"       : module.params['subnet'],
        "gateway"           : module.params['nexthop'],
        "cidrPrefix"        : module.params['prefix'],
        "subinterfaceId"    : -1,
        "wanInterface"      : module.params['interface'],
        "cost"              : module.params['cost'],
        "advertise"         : module.params['advertise'],
        "preferred"         : module.params['preferred'],
        "description"       : module.params['description'],
        "icmpProbeLogicalId": None,
        "sourceIp"          : None,
        "vlanId"            : None
    }
     
    # Check if route is already here
    existing_routes = dev['data']['segments'][0]['routes']['static']
    index = 0
    for existing_route in existing_routes:
        if existing_route['destination'] == module.params['subnet']:
            if module.params['state'] == 'present':
                # Check if route needs to be changed
                if existing_route != route:
                    dev['data']['segments'][0]['routes']['static'][index] = route
                    params = { 'enterpriseId': ent_id, 'id': dev['id'], '_update': dev }
                    update = client.call_api('/configuration/updateConfigurationModule', params)
 
                    module.exit_json(changed=True, argument_spec=module.params, meta=existing_route)
                else:
                    module.exit_json(changed=False, argument_spec=module.params, meta=existing_route)
                break
        index = index+1
 
    if module.params['state'] == 'present':
        dev['data']['segments'][0]['routes']['static'].append(route)
    else:
        dev['data']['segments'][0]['routes']['static'].remove(route)
 
    params = { 'enterpriseId': ent_id, 'id': dev['id'], '_update': dev }
    update = client.call_api('/configuration/updateConfigurationModule', params)
 
    return update
```
Basically, we compare the desired static route against the existing one. If no route exists and ‘state=present’, we can create the route. If the route exists, is not the same one, and ‘state=present’, we can update the route. Finally, if ‘state=absent’, we delete the route.

We just have to update the main function to implement the whole logic and it’s done!

```python
ent = get_enteprise(client, module)
edge = get_edge(client, module, ent['id'])
dev = get_device_module(client, module, ent['id'], edge['id'])
route = add_static_route(client, module, ent['id'], dev)
 
module.exit_json(changed=True, argument_spec=module.params, meta=route)
```

Here is the final code:

```python
from ansible.module_utils.basic import AnsibleModule
import urllib3
 
import client
from client import *
 
def get_enteprise(client, module):
    params = { "networkId": 1, "with": [] }
    enterprises = client.call_api('/network/getNetworkEnterprises', params)
 
    for enterprise in enterprises:
            if enterprise['name'] == module.params['enterprise']:
                return enterprise
                break
    module.fail_json(msg='Cant find enterprise')
 
def get_edge(client, module, ent_id):
    params = { "enterpriseId": ent_id, "with": [] }
    edges = client.call_api('/enterprise/getEnterpriseEdges', params)
 
    for edge in edges:
            if edge['name'] == module.params['edge']:
                return edge
                break
    module.fail_json(msg='Cant find edge')    
 
def get_device_module(client, module, ent_id, edge_id):
    params = { 'enterpriseId': ent_id, 'edgeId': edge_id }
    modules = client.call_api('/edge/getEdgeConfigurationStack', params)
 
    for module in modules[0]['modules']:
            if module['name'] == 'deviceSettings':
                return module
                break
    module.fail_json(msg='Cant find device settings module')
 
def add_static_route(client, module, ent_id, dev):
    route = {
        "destination"       : module.params['subnet'],
        "gateway"           : module.params['nexthop'],
        "cidrPrefix"        : module.params['prefix'],
        "subinterfaceId"    : -1,
        "wanInterface"      : module.params['interface'],
        "cost"              : module.params['cost'],
        "advertise"         : module.params['advertise'],
        "preferred"         : module.params['preferred'],
        "description"       : module.params['description'],
        "icmpProbeLogicalId": None,
        "sourceIp"          : None,
        "vlanId"            : None
    }
     
    # Check if route is already here
    existing_routes = dev['data']['segments'][0]['routes']['static']
    index = 0
    for existing_route in existing_routes:
        if existing_route['destination'] == module.params['subnet']:
            if module.params['state'] == 'present':
                # Check if route needs to be changed
                if existing_route != route:
                    dev['data']['segments'][0]['routes']['static'][index] = route
                    params = { 'enterpriseId': ent_id, 'id': dev['id'], '_update': dev }
                    update = client.call_api('/configuration/updateConfigurationModule', params)
 
                    module.exit_json(changed=True, argument_spec=module.params, meta=existing_route)
                else:
                    module.exit_json(changed=False, argument_spec=module.params, meta=existing_route)
                break
        index = index+1
 
    if module.params['state'] == 'present':
        dev['data']['segments'][0]['routes']['static'].append(route)
    else:
        dev['data']['segments'][0]['routes']['static'].remove(route)
 
    params = { 'enterpriseId': ent_id, 'id': dev['id'], '_update': dev }
    update = client.call_api('/configuration/updateConfigurationModule', params)
 
    return update
 
def main():
    urllib3.disable_warnings()
    module = AnsibleModule(
        argument_spec = dict(
            host                = dict(required=True),
            username            = dict(required=True),
            password            = dict(required=True),
            operator            = dict(required=True, type='bool'),
            enterprise          = dict(required=True),
            edge                = dict(required=True),
            state               = dict(default='present', choices=['present', 'absent']),
            subnet              = dict(required=True),
            prefix              = dict(required=True),
            nexthop             = dict(required=True),
            interface           = dict(required=True),
            cost                = dict(default=0, type='int'),
            advertise           = dict(default=True, type='bool'),
            preferred           = dict(default=True, type='bool'),
            description         = dict(required=False)
        )
    )
 
    try:
        client = VcoRequestManager(hostname=module.params['host'], verify_ssl=False)
        client.authenticate(module.params['username'], module.params['password'], is_operator=module.params['operator'])
    except ApiException as e:
        module.fail_json(msg=e)
 
    ent = get_enteprise(client, module)
    edge = get_edge(client, module, ent['id'])
    dev = get_device_module(client, module, ent['id'], edge['id'])
    route = add_static_route(client, module, ent['id'], dev)
 
    module.exit_json(changed=True, argument_spec=module.params, meta=route)
 
 
if __name__ == '__main__':
    main()
```

For testing, just create a basic Ansible Playbook, such this Yaml File:

```yaml
---
- name: Velocloud Ansible Module
  hosts: localhost
  gather_facts: False
  tasks:
    - name: Static Routes
      velocloud_static_route:
        host: '****'
        username: '****'
        password: '****'
        operator: True
        enterprise: 'test-tenant'
        edge: 'test-edge'
        subnet: "{{ item.subnet }}"
        prefix: "{{ item.prefix }}"
        nexthop: "{{ item.nexthop }}"
        interface: "{{ item.interface }}"
        advertise: "{{ item.advertise }}"
        description: "{{ item.description }}"
        state: absent
      with_items:
        - {subnet: '10.0.0.0', prefix: '24', nexthop: '20.0.0.2', interface: 'GE3', advertise: True, description: 'route1'}
        - {subnet: '20.0.0.0', prefix: '24', nexthop: '20.0.0.1', interface: 'GE3', advertise: False, description: 'route2'}
        - {subnet: '30.0.0.0', prefix: '24', nexthop: '20.0.0.3', interface: 'GE3', advertise: False, description: 'route3'}
        - {subnet: '40.0.0.0', prefix: '24', nexthop: '20.0.0.1', interface: 'GE3', advertise: True, description: 'route4'}
```

And test it with an ansible-playbook command:

```bash
ansible-playbook test.yaml
 
PLAY [Velocloud Ansible Module] **********************************************************************************************************************************************
 
TASK [Static Routes] *********************************************************************************************************************************************************
changed: [localhost] => (item={u'subnet': u'10.0.0.0', u'prefix': u'24', u'description': u'route1', u'interface': u'GE3', u'advertise': True, u'nexthop': u'20.0.0.2'})
changed: [localhost] => (item={u'subnet': u'20.0.0.0', u'prefix': u'24', u'description': u'route2', u'interface': u'GE3', u'advertise': False, u'nexthop': u'20.0.0.1'})
changed: [localhost] => (item={u'subnet': u'30.0.0.0', u'prefix': u'24', u'description': u'route3', u'interface': u'GE3', u'advertise': False, u'nexthop': u'20.0.0.3'})
changed: [localhost] => (item={u'subnet': u'40.0.0.0', u'prefix': u'24', u'description': u'route4', u'interface': u'GE3', u'advertise': True, u'nexthop': u'20.0.0.1'})
[WARNING]: Module did not set no_log for password
 
PLAY RECAP *******************************************************************************************************************************************************************
localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```


