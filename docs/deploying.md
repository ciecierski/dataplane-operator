# Deploying an OpenStackDataPlaneNodeSet

Deploying a dataplane consists of creating the set of OpenStackDataPlaneNodeSet
custom resources that define the layout of the dataplane, and the
OpenStackDataPlaneDeployment custom resources that trigger the ansible
execution to deploy and configure software.

## Samples

The
[config/samples](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/config/samples)
directory contains many sample custom resources that illustrate different
dataplane deployments. These can be used as a starting point and customized for
a given deployment, or a resource can be entirely written from no sample.

## Prerequisites

### ControlPlane

A functional dataplane requires a functional controlplane deployed by
[openstack-operator](https://github.com/openstack-k8s-operators/openstack-operator).
This documentation will make use of resources within the controlplane to
correctly customize the dataplane configuration. The docs here do not cover
deploying a controlplane and assume that has already been completed separately.

### Configured namespace (openstack)

The `oc` commands shown in the documentation assume that the `oc` client has
been configured to use the correct namespace for an OpenStack deployment. By
default, the namespace is `openstack`.

### Review the available fields on the dataplane CRD's

Further documentation on each field available in the dataplane CRD's is
available under the Custom Resources documentation section. This deployment
section does not go into full detail about each available field.

## Deploying a dataplane using pre-provisioned nodes

This section documents using pre-provisioned nodes in the dataplane.
Pre-provisioned nodes already have their OS (operating system) installed and
are powered on and booted into the installed OS.

### Create SSH key secret

Pre-provisioned nodes must be pre-configured with an SSH public key in the
`$HOME/.ssh/authorized_keys` file for a user that has password-less sudo
privileges. Ansible will make use of this user and SSH key when it executes.

The private key for the SSH keypair is created as a Secret in the cluster. Set
the environment variables shown with the name of the secret and the path to the
SSH private key. Run the shown command to create the Secret in the cluster.

    # Name of the secret that will be created in the cluster
    SECRET_NAME="dataplane-ansible-ssh-private-key-secret"
    # Path of the SSH private key file. Public key file should also exist at
    # the same path, but with a ".pub" extension.
    SSH_KEY_FILE="ansibleee-ssh-key-id_rsa"

    oc create secret generic ${SECRET_NAME} \
    --save-config \
    --dry-run=client \
    --from-file=authorized_keys=${SSH_KEY_FILE}.pub \
    --from-file=ssh-privatekey=${SSH_KEY_FILE} \
    --from-file=ssh-publickey=${SSH_KEY_FILE}.pub \
    -o yaml | \
    oc apply -f -

Verify the secret was created:

    oc describe secret dataplane-ansible-ssh-private-key-secret

### Create OpenStackDataPlaneNodeSet

This document will cover writing the `YAML` document for an
`OpenStackDataPlaneNodeSet` resource. Once the document is ready, it will be created
with `oc` as the last step.

Start the `YAML` document in an `openstack-edpm.yaml` file and give the
OpenStackDataPlaneNodeSet a name.

    apiVersion: dataplane.openstack.org/v1beta1
    kind: OpenStackDataPlaneNodeSet
    metadata:
      name: openstack-edpm

An OpenStackDataPlaneNodeSet represents a set of nodes that are configured in a
similar way. Nodes that are deployed with common configurations and that share
a set of common ansible variables can be deployed using the same
OpenStackDataPlaneNodeSet.

Consider using different OpenStackDataPlaneNodeSets to logically group nodes in
a way that makes sense. Differences between nodes are likely to include
configurations due to hardware, location, or networking.  As differences grow,
use different OpenStackDataPlaneNodeSets to manage similarly configured nodes.

The `preProvisioned` field indicates that the nodes are already provisioned,
powered on, and booted into an installed operating system.

A `nodeTemplate` field on the OpenStackDataPlaneNodeSet contains the fields
whose values are inherited by each node in the OpenStackDataPlaneNodeSet.
Within `nodeTemplate`, the fields shown are documented inline in the example.

    apiVersion: dataplane.openstack.org/v1beta1
    kind: OpenStackDataPlaneNodeSet
    metadata:
      name: openstack-edpm
    spec:

      preProvisioned: True

      nodeTemplate:

        # Secret containing the SSH private key used by ansible
        ansibleSSHPrivateKeySecret: dataplane-ansible-ssh-private-key-secret

        ansible:
            # User that has the SSH key for access
            ansibleUser: rhel-user
            # Secret name containing SSH key. Use the same secret name as
            # ${SECRET_NAME} that was used to create the secret.
            ansibleSSHPrivateKeySecret: dataplane-ansible-ssh-private-key-secret

            # Ansible variables that configure how the roles from edpm-ansible
            # customize the deployment.
            ansibleVars:

              # edpm_network_config
              # Default nic config template for a EDPM compute node
              # These vars are edpm_network_config role vars
              edpm_network_config_template: templates/single_nic_vlans/single_nic_vlans.j2

              # See config/samples/dataplane_v1beta1_openstackdataplanenodeset.yaml
              # for the other most common ansible varialbes that need to be set.

The list of ansible variables that can be set under `ansibleVars` is extensive.
To understand what variables are available for each service, see the
documentation in the [Create
OpenStackDataPlaneServices](#create-openstackdataplaneservices) section.

Common configurations that can be enabled with `ansibleVars` are also
documented at [Common Configurations](common_configurations.md).

Add nodes to the dataplane. Each node should have its `role` field set to the
name of its role. Since we are using a single role in this example, that role
name will be `edpm-compute`. Each node will also inherit values
from the `nodeTemplate` field on its role into the `node` field on the node.
However, certain fields will need to be overridden given that they are specific
to a node. In this example, `ansibleVars` has the node specific variables.

Adding nodes to the nodeSet is done under the `spec.Nodes` key, which is a map
with the node names as keys and the values are of type
[NodeSection](openstack_dataplanenodeset.md#nodesection). Nodes within `NodeSection`
can contain the same `ansible` key that also exists in `NodeTemplate`. In this
case, where both are specified, the node specific values override those from
`NodeTemplate`.

---
**NOTE**

In the case of `ansibleVars`, the value is merged with that of the value from
the role. This makes it so that the entire value of `ansibleVars` from the role
does not need to be reproduced for each node just to set a few node specific
values.

---

With the nodes added, the full `openstack-edpm` OpenStackDataPlaneNodeSet
`YAML` document looks like the following:


    apiVersion: dataplane.openstack.org/v1beta1
    kind: OpenStackDataPlaneNodeSet
    metadata:
      name: openstack-edpm
    spec:

      preProvisioned: True

      nodeTemplate:

        # Secret containing the SSH private key used by ansible
        ansibleSSHPrivateKeySecret: dataplane-ansible-ssh-private-key-secret

        ansible:
            # User that has the SSH key for access
            ansibleUser: rhel-user
            # Secret name containing SSH key. Use the same secret name as
            # ${SECRET_NAME} that was used to create the secret.
            ansibleSSHPrivateKeySecret: dataplane-ansible-ssh-private-key-secret

            # Ansible variables that configure how the roles from edpm-ansible
            # customize the deployment.
            ansibleVars:

              # edpm_network_config
              # Default nic config template for a EDPM compute node
              # These vars are edpm_network_config role vars
              edpm_network_config_template: templates/single_nic_vlans/single_nic_vlans.j2

              # See config/samples/dataplane_v1beta1_openstackdataplanenodeset.yaml
              # for the other most common ansible varialbes that need to be set.
      nodes:
        edpm-compute-0:
          hostName: edpm-compute-0
          ansible:
            ansibleHost: 192.168.122.100
            ansibleVars:
              ctlplane_ip: 192.168.122.100
              internalapi_ip: 172.17.0.100
              storage_ip: 172.18.0.100
              tenant_ip: 172.19.0.100
              fqdn_internalapi: edpm-compute-0.example.com
        edpm-compute-1:
          hostName: edpm-compute-1
          ansible:
            ansibleHost: 192.168.122.101
            ansibleVars:
              ctlplane_ip: 192.168.122.101
              internalapi_ip: 172.17.0.101
              storage_ip: 172.18.0.101
              tenant_ip: 172.19.0.101
              fqdn_internalapi: edpm-compute-1.example.com

Create the OpenStackDataPlaneNodeSet using the `oc` command.

    oc create -f openstack-edpm.yaml

Verify that the OpenStackDataPlaneNodeSet is created.

    oc get openstackdataplanenodeset

The output should be similar to:

```console
$ oc get openstackdataplanenodeset
NAME             STATUS   MESSAGE
openstack-edpm   False    Deployment not started
```

### Understanding OpenStackDataPlaneServices

A dataplane is configured with a set of services that define the Ansible plays
or playbooks that are executed to complete the deployment. The
dataplane-operator has a default list of services that are deployed by default
(unless the `services` field is overridden). The default services are provided
within the
[config/services](https://github.com/openstack-k8s-operators/dataplane-operator/tree/main/config/services)
directory.

Each service is a custom resource of type
[OpenStackDataPlaneService](openstack_dataplaneservice.md). The services will
be created and updated automatically during OpenStackDataPlaneNodeSet
reconciliation, when that service is in the list of services for the
OpenStackDataPlaneNodeSet.

See [Composable Services](composable_services.md) for further documentation
about services and customizing services.

Verify the services were created.

    oc get openstackdataplaneservice

The output should be similar to:

    NAME                AGE
    download-cache      6d7h
    configure-network   6d7h
    configure-os        6d6h
    install-os          6d6h
    run-os              6d6h
    validate-network    6d6h
    ovn                 6d6h
    neutron-metadata    6d6h
    libvirt             6d6h
    nova                6d6h
    telemetry           6d6h

Each service uses the
[`playbook`](https://openstack-k8s-operators.github.io/openstack-ansibleee-operator/openstack_ansibleee/#playbook)
or
[`play`](https://openstack-k8s-operators.github.io/openstack-ansibleee-operator/openstack_ansibleee/#play)
field from the `OpenStackAnsibleEE` CRD provided
by
[openstack-ansibleee-operator](https://github.com/openstack-k8s-operators/openstack-ansibleee-operator)
to define the Ansible execution for the service.

For example, the playbooks for the `install-os` service can be seen by
describing the resource.

    oc describe openstackdataplaneservice install-os

Any playbook listed in the `osp.edpm` namespace is provided by the
[edpm-ansible](https://github.com/openstack-k8s-operators/edpm-ansible)
project. Within that project, the ansible variables that can be used to
configure the role are documented.

For example, in the describe output for the `install-os` service, the
`osp.edpm.install_os` playbook is seen.

The playbooks are available at
<https://github.com/openstack-k8s-operators/edpm-ansible/tree/main/playbooks>.

---
**NOTE**

If the default provided services are edited, those edits will be lost after any
further role reconciliations.

---

### Deploy the dataplane

With the OpenStackDataPlaneNodeSet resources created, it can be seen from their
status message that they have not yet been deployed. This means no ansible has
been executed to configure any of the services on the nodes. They still need to
be deployed.

To deploy the `openstack-edpm` OpenStackDataPlaneNodeSet resource, an
OpenStackDataPlaneDeployment resource must be created. An
OpenStackDataPlaneDeployment works similarly to a [Kubernetes
Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/), in that
is triggers and continues to reconcile the resources necessary to complete the
ansible execution for the OpenStackDataPlaneNodeSet until it succeeds. Once
succeeded, the OpenStackDataPlaneDeployment is considered complete. To start
another ansible execution, another OpenStackDataPlaneDeployment resource is
created. This allows for tracking the history of all the ansible executions
that have been completed for the dataplane.

Create an OpenStackDataPlaneDeployment resource with the following sample
`YAML`.

    apiVersion: dataplane.openstack.org/v1beta1
    kind: OpenStackDataPlaneDeployment
    metadata:
      name: openstack-edpm
    spec:

      # ansible related fields that control the ansible execution
      ansibleTags: ""
      ansibleLimit: ""
      ansibleSkipTags: ""

      # List of OpenStackDataPlaneNodeSet names included in the ansible
      # execution.
      nodeSets:
        - openstack-edpm

With the OpenStackDataPlaneDeployment started, ansible will be executed to
configure the nodes within the list OpenStackDataPlaneNodeSets. List additional
OpenStackDataPlaneNodeSets within the `nodeSets` list to deploy
OpenStackDataPlaneNodeSets simultaneously. When the deployment is complete, the
status messages will change to indicate the deployment is ready.

```console
$ oc get openstackdataplanenodeset
NAME             STATUS   MESSAGE
openstack-edpm   True     Ready
```

If the deployment involved adding new compute nodes then after the deployment
is ready those compute nodes need to be mapped in nova. To do that run the
following command:
```console
oc rsh nova-cell0-conductor-0 nova-manage cell_v2 discover_hosts --verbose
```

#### Service specific configurations

Some of the predefined OpenStackDataPlaneService CRs requires manual
configuration before the deployment can be started.

* `OpenStackDataPlaneService/nova`
  * The service needs an SSH key-pair provided. Generate an ssh key-pair and store it in a Secret named `nova-migration-ssh-key`.
    ```console
    $ cd "$(mktemp -d)"
    $ ssh-keygen -f ./id -t ecdsa-sha2-nistp521 -N ''
    $ oc create secret generic nova-migration-ssh-key \
    -n openstack \
    --from-file=ssh-privatekey=id \
    --from-file=ssh-publickey=id.pub \
    --type kubernetes.io/ssh-auth
    $ rm id*
    $ cd -
    ```

#### Overriding services for the deployment

The list of services that will be deployed when an
`OpenStackDataPlaneDeployment` is created is set on each
`OpenStackDataPlaneNodeSet` that is included in the `nodeSets` list on the
`OpenStackDataPlaneDeployment`.

This allows for deploying a different set of services on different
`OpenStackDataPlaneNodeSets` using the same `OpenStackDataPlaneDeployment`
resource simultaneously. It also maintains the association between services and
nodeSets on the nodeSet itself. This association is important when nodeSets are
used to group nodes with hardware and configuration differences that require
the need for deploying different services on different nodeSets.

In some specific cases, it may be needed to override what services are deployed
on all nodeSets included in an `OpenStackDataPlaneDeployment`. These cases can
vary, but are often related to day 2 workflows such as update, upgrade, and
scale out. In these cases, it may be needed to execute a smaller subset of
services, or just a single service, across all nodeSets in the
`OpenStackDataPlaneDeployment`.

The `servicesOverride` field on `OpenStackDataPlaneDeployment` allow for this
behavior. Setting this field changes what services are deployed when the
`OpenStackDataPlaneDeployment` is created. If the field is set, only the
services listed in the field will be deployed on all nodeSets.

The following example `OpenStackDataPlaneDeployment` resource illustrates using
`servicesOverride` to perform a pre-upgrade task of executing just the ovn
service.

    apiVersion: dataplane.openstack.org/v1beta1
    kind: OpenStackDataPlaneDeployment
    metadata:
      name: openstack-edpm-pre-upgrade-ovn
    spec:

      nodeSets:
        - openstack-edpm

      // Only the services here will be executed. Overriding any services value
      // on the openstack-edpm nodeSet.
      servicesOverride:
        - ovn

### Understanding dataplane conditions

Each dataplane resource has a series of conditions within their `status`
subresource that indicate the overall state of the resource, including its
deployment progress.

Example `OpenStackDataPlaneNodeSet` resource conditions:

```console
$ oc get openstackdataplanenodeset openstack-edpm -o json | jq .status.conditions[].type
"Ready"
"DeploymentReady"
"SetupReady"
```

#### Condition Progress

The `Ready` condition reflects the latest condition state that has changed.
For an `OpenStackDataPlaneNodeSet`, until an `OpenStackDataPlaneDeployment` has
been started and finished successfully, the `Ready` condition will be `False`.
When the deployment succeeds, it will be set to `True`. A subsequent deployment
that is started will set the condition back to `False` until the deployment
succeeds when it will be set back to `True`.

`SetupReady` will be set to `True` once all setup related tasks for a resource
are complete. Setup related tasks include verifying the SSH key secret and
verifying other fields on the resource, as well as creating the Ansible
inventory for each resource.

The conditions are futher detailed below.

#### OpenStackDataPlaneNodeSet Conditions and Status

OpenStackDataPlaneNodeSet uses the following conditions:

| Condition Type | Description |
| --- | --- |
| Ready | True when the NodeSet has been deployed successfully. False when the deployment is not yet requested or failed, or there are other failed conditions. |
| DeploymentReady | True when the the NodeSet has been deployed successfully |
| InputReady | True when the required inputs are available and ready |
| SetupReady | True when all related setup tasks are completed |
| NodeSetIPReservationReady | True when the IPSet resources are ready |
| NodeSetDNSDataReady | True when DNSData resources are ready |
| NodeSetBaremetalProvisionReady | True when baremetal hosts are provisioned and ready |

OpenStackDataPlaneNodeSet has the following status fields:

| Status Field | Description |
| --- | --- |
| Deployed | Boolean indicating successful deployment |
| DNSClusterAddresses | |
| CtlplaneSearchDomain | |

#### OpenStackDataPlaneDeployment Conditions and Status

| Condition Type | Description |
| --- | --- |
| Ready | True when the deployment has succeeded. False when the deployment has failed, or there are other failed conditions. |
| DeploymentReady | True when the deployment has succeeded |
| InputReady | True when the required inputs are available and ready |
| <NodeSet> Deployment Ready | True when the deployment has succeeded for the named <NodeSet>, indicating all services for the <NodeSet> have succeeded |
| <NodeSet> <Service> Deployment Ready | True when the deployment has succeeded for the named <NodeSet> and <Service> |

Each `<NodeSet> <Service> Deployment Ready` specific condition will be set to
`True` as that service completes successfully for the named `<NodeSet>`. Once
all services are complete for a `<NodeSet>`, the `<NodeSet> Deployment Ready`
will be set to `True`.  Looking at the service conditions will indicate which
services have completed their deployment, or in failure cases, which services
failed and for which NodeSets.

OpenStackDataPlaneDeployment has the following status fields:

| Status Field | Description |
| --- | --- |
| Deployed | Boolean indicating successful deployment. All Services for all NodeSets have succeeded. |


#### OpenStackDataPlaneService Conditions and Status

| Condition Type | Description |
| --- | --- |
| Ready | True when the service has been created and is ready for use |

OpenStackDataPlaneService has no additional Status fields.

Each service specific condition will be set to `True` as that service completes
successfully. Looking at the service conditions will indicate which services
have completed their deployment, or in failure cases, which services failed.

### OpenStackDataPlaneNodeSet deployment hashes

Each `OpenStackDataPlaneService` can optionally have an associated list of
`ConfigMaps` and `Secrets` that are mounted as file data into the
`OpenStackAnsibleEE` job started to deploy that service. The ansible content
then is able to consume those files as necessary. See [Configuring a custom
service](composable_services.md#configuring-a-custom-service) for more details.

When an `OpenStackDataPlaneDeployment` succeeds, the computed hash of each
`ConfigMap` and `Secret` for each `OpenStackDataPlaneService` that was deployed
is saved on the status of each `OpenStackDataPlaneNodeSet` referenced by the
`OpenStackDataPlaneDeployment`.

These hashes can be compared against the current hash of the `ConfigMap` or
`Secret` to see if there is newer input data that has not been deployed to the
`OpenStackDataPlaneNodeSet`. For example if the hash of
`nova-cell1-compute-config` `Secret` in the `OpenStackDataPlaneNodeSet` status
is different from the hash of `nova-cell1-compute-config` in the
`novacell/nova-cell1` status, then there is nova-compute control plane configuration
data the needs to be deployed to the EDPM compute nodes.

For example, the following hashes are saved on the `OpenStackDataPlaneNodeSet`
status after a typical deployment:

```console
$ oc get openstackdataplanenodeset openstack-edpm -o yaml

<snip>
status:
  conditions:
	<snip>
  configMapHashes:
    ovncontroller-config: n655h5...
  secretHashes:
    neutron-dhcp-agent-neutron-config: n9ch5...
    neutron-ovn-metadata-agent-neutron-config: n588h...
    neutron-sriov-agent-neutron-config: n648h...
    nova-cell1-compute-config: n576h...
    nova-metadata-neutron-config: n56fh...
```
