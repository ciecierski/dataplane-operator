# Inheritance

An `node` inherits any attribute of an
`nodeTemplate` but those attributes may also be overridden
on the node level.

Suppose the following CR is created with `oc create -f
openstackdataplanerole-sample.yaml`:

```yaml
---
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneRole
metadata:
  name: openstackdataplanerole-sample
spec:
  dataPlaneNodes:
  - name: openstackdataplanenode-sample-1
    ansibleHost: 192.168.122.18
    hostName: openstackdataplanenode-sample-1.localdomain
    node:
      networks:
      - fixedIP: 192.168.122.18
        network: ctlplane
  - name: openstackdataplanenode-sample-2
    ansibleHost: 192.168.122.19
    hostName: openstackdataplanenode-sample-2.localdomain
    node:
      networks:
      - fixedIP: 192.168.122.19
        network: ctlplane
  nodeTemplate:
    ansiblePort: 22
    ansibleUser: cloud-admin
    managed: false
    managementNetwork: ctlplane
    networkConfig:
      template: templates/net_config_bridge.j2
```

Because of inheritance, redundant information did not need to be
provided to both nodes. Only the information which differed per node,
e.g. `ansibleHost`, had to be specified. Furthermore, the redundant
information is not seen in the two nodes' CRs. I.e. we do not see the
following from the `nodeTemplate` in node 1 and 2 above.

```yaml
    ansiblePort: 22
    ansibleUser: cloud-admin
    managed: false
    managementNetwork: ctlplane
    networkConfig:
      template: templates/net_config_bridge.j2
```

However, it's unambiguous that each node has `ansiblePort` 22
because they have `role: openstackdataplanerole-sample`. If the
node is inspected however, port 22 will be set.

Almost any top level property in a node overrides the whole property
in a nodeTemplate. E.g. if the nodeset `nodeTemplate` had a list like the
following:

```
    foo:
      - bar: baz
```

and a node had a list like the following:

```
    foo:
      - qux: quux
```

Then the node will only have `{"qux": "quux"}` in its list. I.e. there
would not be any merging and the node would not also have `{"bar":
"baz"}` in its list. The one exception to this rule is the
`AnsibleVars` property. If the following CRs are created:

```yaml
kind: OpenStackDataPlaneRole
metadata:
  name: edpm-compute
spec:
  nodeTemplate:
    ansibleVars:
      neutron_public_interface_name: eth0
      timesync_ntp_servers:
        - hostname: clock.redhat.com
        - hostname: clock2.redhat.com
  ...
```

```yaml
kind: OpenStackDataPlaneNode
metadata:
  name: edpm-compute-0
spec:
  role: edpm-compute
  node:
    ansibleVars:
      tenant_ip: 192.168.24.100
  ...
```

The `ConfigMap` containing the inventory would have the following
vars:

```yaml
      neutron_public_interface_name: eth0
      tenant_ip: 192.168.24.100
      timesync_ntp_servers:
        - hostname: clock.redhat.com
        - hostname: clock2.redhat.com
```

Only top level keys are compared for merging. E.g. if `edpm-compute-0`
had a `timesync_ntp_servers` list with `hostname: clock3.redhat.com`, then
the resultant inventory for the node would not have three NTP servers;
it would only have `clock3.redhat.com`. I.e. there is no "deep
merging".
