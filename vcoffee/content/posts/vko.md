---
title: "A VELOCLOUD KUBERNETES OPERATOR !"
date: 2021-01-15T17:07:15+01:00
draft: false
---


### *DISCLAIMER: The views and opinions expressed on this blog are our own and may not reflect the views and opinions of our employer*



As Kubernetes is becoming the de-facto standard to run the new modern applications, there is a need to include the SDWAN configuration as part of the application development.

Let’s imagine a dev coding a new app. As part of the application testing, it can be useful to adapt the SDWAN policy, such as bandwidth, the QoS settings, the link steering and so on. The dev could use the same YAML files and kubectl commands that he loves and just include the SDWAN parameters.

Obviously, the settings that the dev could use should be limited to a range that has been defined by infra admins. Some namespaces could set more bandwidth than others, for example. The link steering policy could be limited to “Internet Only” for pre-prod namespaces, as another example.

In this post, I’m going to cover the creation of such integration. We’re going to create a Velocloud CRD (Custom Resource Definition), that the infra admins will be able to use to define the set of settings allowed for a dev. Then, we’ll create a Kubernetes Operator, which is going to talk with the Velocloud Orchestrator to implement the SDWAN logic. Finally, we’ll define some YAML samples to demonstrate how a dev can use such a Velocloud CRD.

So we need 2 things to start coding: a Velocloud Orchestrator and a Kubernetes 😉

For Kubernetes, I will use Kind as a very simple Kubernetes running on my laptop. The setup is really easy to do, and you can have a running Kube on your laptop after only a few minutes: https://kind.sigs.k8s.io/docs/user/quick-start/

I’m going the create 2 namespaces:

```bash
adeleporte@adeleporte-a01 ns1 % kubectl create ns ns1
namespace/ns1 created
adeleporte@adeleporte-a01 ns1 % kubectl create ns ns2
namespace/ns2 created
adeleporte@adeleporte-a01 ns1 % kubectl get ns
NAME                 STATUS   AGE
default              Active   8m27s
kube-node-lease      Active   8m29s
kube-public          Active   8m29s
kube-system          Active   8m29s
local-path-storage   Active   8m23s
ns1                  Active   6s
ns2                  Active   5s
```

For the Velocloud Orchestrator, I will use a new feature, the API tokens. Once logged as a Administrator, you can now create/revoke some API tokens, which seems to be a good idea in my own opinion.

Velocloud CRD definition
Now, let’s start coding a very basic Velocloud CRD in a crd.yaml file:

```yaml
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
    name: velocloudbps.vcn.cloud
spec:
    scope: Namespaced
    group: vcn.cloud
    version: v1
    names:
      kind: VelocloudBP
      plural: velocloudbps
      singular: velocloudbp
      shortNames:
      - velo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
    name: velocloud-group-admin
rules:
- apiGroups:
  - vcn.cloud
  resources:
  - velocloudbps
  - velocloudbps/finalizers
  verbs: [ get, list, create, update, delete, deletecollection, watch ]
 
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
    name: velocloud-group-rbac
subjects:
- kind: ServiceAccount
  name: default
  #namespace: default
roleRef:
    kind: Role
    name: velocloud-group-admin
    apiGroup: rbac.authorization.k8s.io
```

```bash
adeleporte@adeleporte-a01 ns1 % kubectl apply -f crd.yaml -n ns1
customresourcedefinition.apiextensions.k8s.io/velocloudbps.vcn.cloud created
```

Here is what we’ve done:

We have a new kind of CRD, called velocloudbps + a new role, called velocloud-group-admin, which can use get, list, create…watch commands, and this role in bind to the default service account.

By doing so, we now have some new kubectl commands:-)

```bash
adeleporte@adeleporte-a01 ns1 % kubectl get velocloudbps
No resources found.

adeleporte@adeleporte-a01 ns1 % kubectl get velocloudbps
No resources found.
adeleporte@adeleporte-a01 ns1 % kubectl get velo
No resources found.
```

Ok, not very useful at this stage, but we have our new Velocloud CRD! Let’s modify the initial crd.yaml file to include some validation. This will enable us (and the infra admins) to define which parameter can be set on this CRD. Let’s say that we want to define the Velocloud Configuration Profile that is going to be used, the FQDN of the application currently being developed by the dev, the TCP port, and the Business Policy settings (Link steering policy, QoS priority, Bandwidth Limit). We also define the column to be displayed when using commands such as “kubectl get velocloudbps -o wide”

```yaml
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
    name: velocloudbps.vcn.cloud
spec:
    scope: Namespaced
    group: vcn.cloud
    version: v1
    names:
      kind: VelocloudBP
      plural: velocloudbps
      singular: velocloudbp
      shortNames:
      - velo
    validation:
        openAPIV3Schema:
          properties:
            spec:
              type: object
              properties:
                profile:
                    type: string
                    description: profile
                    enum:
                    - Quick Start Profile
                name:
                    type: string
                    description: name
                fqdn:
                    type: string
                    description: fqdn
                    pattern: '^(.*).vcn.cloud$'
                dport:
                    type: integer
                    description: dport
                service-class:
                    type: string
                    description: sc
                    enum:
                    - realtime
                    - transactional
                    - bulk
                link-policy:
                    type: string
                    description: lp
                    enum:
                    - auto
                    - fixed
                service-group:
                    type: string
                    description: sg
                    enum:
                    - PRIVATE_WIRED
                    - PUBLIC_WIRED
                    - ALL
                priority:
                    type: string
                    description: priority
                    enum:
                    - high
                    - normal
                    - low
                bandwidth-limit:
                    type: integer
                    description: bpl
                    minimum: -1
                    maximum: 50
additionalPrinterColumns:
      - name: Profile
        type: string
        description: The profile
        JSONPath: .spec.profile
      - name: FQDN
        type: string
        description: The fqdn
        JSONPath: .spec.fqdn
      - name: Service Class
        type: string
        priority: 1
        description: The service class
        JSONPath: .spec.service-class
      - name: Priority
        type: string
        priority: 1
        description: The priority
        JSONPath: .spec.priority
      - name: Link Policy
        type: string
        priority: 1
        description: The link Policy
        JSONPath: .spec.link-policy
      - name: Bandwidth Limit
        type: integer
        priority: 1
        description: The bandwidth Limit
        JSONPath: .spec.bandwidth-limit
```

# Velocloud Kubernetes Operator
Finally, we need now is an operator. Basically, we want Kubernetes to run our Velocloud operator when a CRD is created/updated/deleted. For that, we’ll create a deployment of a pod with 2 containers: the operator itself (which is going to talk to the Velocloud Orchestrator) and a kubectl proxy (to handle authentication).

Let’s change again our crd.yaml file to include this deployment as part of the CRD definition:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
    name: operator
    labels:
      app: operator
spec:
    selector:
      matchLabels:
        app: operator
    template:
      metadata:
        labels:
          app: operator
      spec:
        containers:
        - name: proxycontainer
          image: lachlanevenson/k8s-kubectl
          command: ["kubectl","proxy","--port=8001"]
        - name: app
          image: adeleporte/velokube
          env:
          - name: res_namespace
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: VCO_URL
            value: https://vco22-fra1.velocloud.net/portal/rest
          - name: VCO_TOKEN
            value: changeme
          - name: verbose
            value: log
```

```bash
adeleporte@adeleporte-a01 ns1 % kubectl apply -f crd.yaml -n ns1
customresourcedefinition.apiextensions.k8s.io/velocloudbps.vcn.cloud created
```

At this stage, the Velocloud Kubernetes operator logic is defined. The adeleporte/velokube container (we’ll discuss about this container right after) is responsible for watching at velocloudbps CRD changes. This container will then implement the necessary changes over the Velocloud Orchestrator. Let’s now deep in the adeleporte/velokube container:

```python
import requests
import warnings
import json
import os
from copy import deepcopy
import logging
import sys
 
# Env variables
vco_url = os.getenv("VCO_URL")
token = os.getenv("VCO_TOKEN")
verbose = os.getenv("verbose")
 
base_url = "http://127.0.0.1:8001"
namespace = os.getenv("res_namespace", "default")
 
# Logging
log = logging.getLogger(__name__)
out_hdlr = logging.StreamHandler(sys.stdout)
out_hdlr.setFormatter(logging.Formatter('%(asctime)s %(message)s'))
out_hdlr.setLevel(logging.INFO)
log.addHandler(out_hdlr)
log.setLevel(logging.INFO)
 
 
def client(method, body):
  warnings.filterwarnings('ignore', message='Unverified HTTPS request')
  headers = {'Content-Type': 'application/json', 'Authorization': 'Token {}'.format(token)}
  try:
    resp = requests.post('{}/{}'.format(vco_url, method), data=json.dumps(body), headers=headers, verify=False)
    if (resp.status_code == 200):
      resp_parsed = json.loads(resp.text)
      return resp_parsed
    else:
      log.info(resp.status_code)
      return False
  except:
    log.info('Request failed')
    return False
 
 
def GetConfiguration(profilename):
 
  profiles = client('enterprise/getEnterpriseConfigurationsPolicies', {})
  for profile in profiles:
    if profile['name'] == profilename:
      config = client('configuration/getConfiguration', {"id": profile['id'], "with": ["modules"]})
      return config
   
  log.info('Failed to find Profile')
  return False
 
def GetQosModule(config):
  for module in config['modules']:
    if module['name'] == "QOS":
      qos_module = module
      return qos_module
      break
  log.info('Cant find Qos Module')
  return None
 
def DeleteQosRule(qos_module, spec):
  # Look for rule to delete
  index = 0
  for rule in qos_module['data']['segments'][0]['rules']:
    if rule['name'] == spec['name']:
      qos_module['data']['segments'][0]['rules'].pop(index)
    index = index+1
   
  # Update the module
  update = client('configuration/updateConfigurationModule', {"id": qos_module['id'], "_update": {"name": "QOS", "data": qos_module['data']}})
 
  log.info('rule deleted')
  return update
 
def SetQosRule(qos_module, spec):
  template_rule = None
  for rule in qos_module['data']['segments'][0]['defaults']:
    if rule['name'] == "Default-Any-Other":
      template_rule = rule
      break
   
  if (template_rule == None):
    log.info('Default rule Default-Any-Other not found')
    return False
 
  new_rule = deepcopy(template_rule)
 
  # Name
  new_rule['name'] = spec['name']
 
  # Match
  new_rule['match']['hostname'] = spec['fqdn']
  new_rule['match']['dport_high'] = spec['dport']
  new_rule['match']['dport_low'] = spec['dport']
  new_rule['match']['proto'] = 6
 
  # Service Class
  new_rule['action']['QoS']['type'] = spec['service-class']
 
  # Priority
  new_rule['action']['QoS']['rxScheduler']['priority'] = spec['priority']
  new_rule['action']['QoS']['txScheduler']['priority'] = spec['priority']
 
  # Bandwidth Limit
  new_rule['action']['QoS']['rxScheduler']['bandwidthCapPct'] = spec['bandwidth-limit']
  new_rule['action']['QoS']['txScheduler']['bandwidthCapPct'] = spec['bandwidth-limit']
   
  # Link Steering
  new_rule['action']['edge2CloudRouteAction']['linkPolicy'] = spec['link-policy']
  new_rule['action']['edge2CloudRouteAction']['serviceGroup'] = spec['service-group']
  new_rule['action']['edge2DataCenterRouteAction']['linkPolicy'] = spec['link-policy']
  new_rule['action']['edge2DataCenterRouteAction']['serviceGroup'] = spec['service-group']
  new_rule['action']['edge2EdgeRouteAction']['linkPolicy'] = spec['link-policy']
  new_rule['action']['edge2EdgeRouteAction']['serviceGroup'] = spec['service-group']
 
  # Look if rule already exists
  index = 0
  for rule in qos_module['data']['segments'][0]['rules']:
    if rule['name'] == spec['name']:
      qos_module['data']['segments'][0]['rules'][index] = new_rule
      update = client('configuration/updateConfigurationModule', {"id": qos_module['id'], "_update": {"name": "QOS", "data": qos_module['data']}})
      log.info('rule updated')
      return update
    index = index+1
 
  # Rule not found, insert it at the top
  qos_module['data']['segments'][0]['rules'].insert(0, new_rule)
  update = client('configuration/updateConfigurationModule', {"id": qos_module['id'], "_update": {"name": "QOS", "data": qos_module['data']}})
  log.info('rule added')
  return update
 
 
def main():
  config = GetConfiguration()
  qos_module = GetQosModule(config)
  update = SetQosRule(qos_module, fqdn)
 
 
def event_loop():
    log.info("Starting the service")
    url = '{}/apis/vcn.cloud/v1/namespaces/{}/velocloudbps?watch=true"'.format(
        base_url, namespace)
    r = requests.get(url, stream=True)
    # We issue the request to the API endpoint and keep the conenction open
    for line in r.iter_lines():
        obj = json.loads(line)
        # We examine the type part of the object to see if it is MODIFIED
        if (verbose == 'log'):
          log.info(obj)
 
        config = GetConfiguration(obj['object']['spec']['profile'])
        qos_module = GetQosModule(config)
 
        if (obj['type'] == 'ADDED'):
          SetQosRule(qos_module, obj['object']['spec'])
        if (obj['type'] == 'MODIFIED'):
          SetQosRule(qos_module, obj['object']['spec'])
        if (obj['type'] == 'DELETED'):
          DeleteQosRule(qos_module, obj['object']['spec'])
 
 
event_loop()
```

The code is written is Python. The interesting parts are:

* we use a Stream Request to watch at velocloudbps events (/apis/vcn.cloud/v1/namespaces/{}/velocloudbps?watch=true)
* this socket acts as a loop and generate a line for each event
* when a new event arises, we decode the details (obj = json.loads(line))
* depending of the type of changes (creation/modification/deletion) (obj[‘type’]), we use a different Python function
* the Velocloud Business Profile is stored inside a Configuration module (the profile) and a Qos module (the business policies), so we have a implement some code to fetch these values
* in case of creation, we copy the default QoS rule, change some parameters according to the CRD, and update the QoS module

This code is copied in a Python container and uploaded to docker hub. Here is the docker file to generate the adeleporte/velokube container:

```bash
FROM python:latest
RUN pip install requests
COPY operator/main.py /main.py
ENTRYPOINT [ "python", "/main.py" ]
```

If everything works as expected, you should see your deployment as ok in Kube:

```bash
adeleporte@adeleporte-a01 ns1 % kubectl get deployments -n ns1
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
operator   1/1     1            1           96m
adeleporte@adeleporte-a01 ns1 % kubectl describe deployment operator -n ns1
Name:                   operator
Namespace:              ns1
CreationTimestamp:      Tue, 21 Jul 2020 14:02:31 +0200
Labels:                 app=operator
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"operator"},"name":"operator","namespace":"ns1"},...
Selector:               app=operator
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=operator
  Containers:
   proxycontainer:
    Image:      lachlanevenson/k8s-kubectl
    Port:       <none>
    Host Port:  <none>
    Command:
      kubectl
      proxy
      --port=8001
    Environment:  <none>
    Mounts:       <none>
   app:
    Image:      adeleporte/velokube
    Port:       <none>
    Host Port:  <none>
    Environment:
      res_namespace:   (v1:metadata.namespace)
      VCO_URL:        https://vco22-fra1.velocloud.net/portal/rest
      VCO_TOKEN:      changeme
      verbose:        log
    Mounts:           <none>
  Volumes:            <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   operator-5bf4b944cd (1/1 replicas created)
Events:          <none>
adeleporte@adeleporte-a01 ns1 %
adeleporte@adeleporte-a01 ns1 % kubectl get pods -n ns1   
NAME                        READY   STATUS    RESTARTS   AGE
operator-5bf4b944cd-tnfh9   2/2     Running   2          98m
adeleporte@adeleporte-a01 ns1 % 
adeleporte@adeleporte-a01 ns1 % kubectl logs operator-5bf4b944cd-tnfh9 app -n ns1
2020-07-21 13:14:51,674 Starting the service
adeleporte@adeleporte-a01 ns1 %
```


# Using the Velocloud Kubernetes Operator
Everything is done! Let’s test our new Velocloud Kubernetes Operator 😉 
So now, I’ll acting as a dev, let’s create a velocloud.yaml as part of my application manifest:

```yaml
---
apiVersion: vcn.cloud/v1
kind: VelocloudBP
metadata:
    name: velokube1
spec:
    profile: Quick Start Profile
    name: velokube1
    fqdn: velokube1.vcn.cloud
    dport: 8080
    service-class: realtime
    link-policy: auto
    service-group: ALL
    priority: high
    bandwidth-limit: -1
---
apiVersion: vcn.cloud/v1
kind: VelocloudBP
metadata:
    name: velokube2
spec:
    profile: Quick Start Profile
    name: velokube2
    fqdn: velokube2.vcn.cloud
    dport: 443
    service-class: realtime
    link-policy: fixed
    service-group: PUBLIC_WIRED
    priority: high
    bandwidth-limit: 40
```

```bash
adeleporte@adeleporte-a01 ns1 % kubectl create -f velocloud.yaml --namespace ns1
velocloudbp.vcn.cloud/velokube1 created
velocloudbp.vcn.cloud/velokube2 created
adeleporte@adeleporte-a01 ns1 % 
adeleporte@adeleporte-a01 ns1 % kubectl get velocloudbps -n ns1
NAME        PROFILE               FQDN
velokube1   Quick Start Profile   velokube1.vcn.cloud
velokube2   Quick Start Profile   velokube2.vcn.cloud
adeleporte@adeleporte-a01 ns1 % kubectl get velocloudbps -n ns1 -o wide
NAME        PROFILE               FQDN                  SERVICE CLASS   PRIORITY   LINK POLICY   BANDWIDTH LIMIT
velokube1   Quick Start Profile   velokube1.vcn.cloud   realtime        high       auto          -1
velokube2   Quick Start Profile   velokube2.vcn.cloud   realtime        high       fixed         40
adeleporte@adeleporte-a01 ns1 % 
```html

For my application velokube1.vcn.cloud, I want to have a RealTime Service Class, use all the available links, have a high QoS priority and no bandwidth limit.

For my application velokube2.vcn.cloud, I want to have a RealTime Service Class, use only Internet Links, have a High QoS priority and a bandwidth limit set to 40%.

Let’s check the Velocloud configuration:

And the operator logs

```bash
deleporte@adeleporte-a01 ns1 % kubectl logs operator-5bf4b944cd-tnfh9 app -n ns1
2020-07-21 13:14:51,674 Starting the service
2020-07-21 13:47:05,250 {'type': 'ADDED', 'object': {'apiVersion': 'vcn.cloud/v1', 'kind': 'VelocloudBP', 'metadata': {'creationTimestamp': '2020-07-21T13:47:05Z', 'generation': 1, 'name': 'velokube1', 'namespace': 'ns1', 'resourceVersion': '18703', 'selfLink': '/apis/vcn.cloud/v1/namespaces/ns1/velocloudbps/velokube1', 'uid': 'b2767e70-642e-4e4b-aad8-cd78fd089b7b'}, 'spec': {'bandwidth-limit': -1, 'dport': 8080, 'fqdn': 'velokube1.vcn.cloud', 'link-policy': 'auto', 'name': 'velokube1', 'priority': 'high', 'profile': 'Quick Start Profile', 'service-class': 'realtime', 'service-group': 'ALL'}}}
2020-07-21 13:47:05,629 rule added
2020-07-21 13:47:05,629 {'type': 'ADDED', 'object': {'apiVersion': 'vcn.cloud/v1', 'kind': 'VelocloudBP', 'metadata': {'creationTimestamp': '2020-07-21T13:47:05Z', 'generation': 1, 'name': 'velokube2', 'namespace': 'ns1', 'resourceVersion': '18704', 'selfLink': '/apis/vcn.cloud/v1/namespaces/ns1/velocloudbps/velokube2', 'uid': '646346aa-d8b5-4ef3-bbb0-c8fb0ff7b062'}, 'spec': {'bandwidth-limit': 40, 'dport': 443, 'fqdn': 'velokube2.vcn.cloud', 'link-policy': 'fixed', 'name': 'velokube2', 'priority': 'high', 'profile': 'Quick Start Profile', 'service-class': 'realtime', 'service-group': 'PUBLIC_WIRED'}}}
2020-07-21 13:47:05,997 rule added
adeleporte@adeleporte-a01 ns1 %
```

Now let’s imagine that I want to change my mind for application2. I change some settings in the YAML file (Link Policy Public to Private, Priority High to Low, Bandwidth Limit from 40 to 10)

```yaml
---
apiVersion: vcn.cloud/v1
kind: VelocloudBP
metadata:
    name: velokube2
spec:
    profile: Quick Start Profile
    name: velokube2
    fqdn: velokube2.vcn.cloud
    dport: 443
    service-class: realtime
    link-policy: fixed
    service-group: PRIVATE_WIRED
    priority: low
    bandwidth-limit: 10

adeleporte@adeleporte-a01 ns1 % kubectl apply -f velocloud.yaml --namespace ns1
velocloudbp.vcn.cloud/velokube1 unchanged
velocloudbp.vcn.cloud/velokube2 configured
adeleporte@adeleporte-a01 ns1 %

Set some limits to devs
Now, let’s imagine that I want to change the bandwidth limit to 75%

---
apiVersion: vcn.cloud/v1
kind: VelocloudBP
metadata:
    name: velokube2
spec:
    profile: Quick Start Profile
    name: velokube2
    fqdn: velokube2.vcn.cloud
    dport: 443
    service-class: realtime
    link-policy: fixed
    service-group: PRIVATE_WIRED
    priority: low
    bandwidth-limit: 75

velocloudbp.vcn.cloud/velokube1 unchanged
The VelocloudBP "velokube2" is invalid: spec.bandwidth-limit: Invalid value: 50: spec.bandwidth-limit in body should be less than or equal to 50
adeleporte@adeleporte-a01 ns1 %
```

As a developer, I’m not allowed to set more than 50% of bandwidth, because this limit has been set by the infra team when defining the CRD.

As a result, the infra team can define very specific limit for each namespace and developers are free to set the Velocloud configutation within these limit.

# Conclusion
I hope that this post will give you some insights about what can be achieved when integrating such great solutions together, like Velocloud and Kubernetes. The infra/network/wan teams can now work together with the dev team to align infrastructure needs and business/dev agility.



