Planning the new deployment
===========================

Just like you did back when you installed your Director deployed OpenStack, the
upgrade/migration to the podified OpenStack requires planning various aspects
of the environment such as node roles, planning your network topology, and
storage.

In this document we cover some of this planning, but it is recommended to read
the whole adoption guide before actually starting the process to be sure that
there is a global understanding of the whole process.

## Configurations

There is a fundamental difference between the Director and Operator deployments
regarding the configuration of the services.

In Director deployments many of the service configurations are abstracted by
Director specific configuration options. A single Director option may trigger
changes for multiple services and support for drivers (for example Cinder's)
required patches to the Director code base.

In Operator deployments this has changed to what we believe is a simpler
approach: reduce the installer specific knowledge and leverage OpenShift and
OpenStack service specific knowledge whenever possible.

To this effect OpenStack services will have sensible defaults for OpenShift
deployments and human operators will provide configuration snippets to provide
necessary configuration, such as cinder backend configuration, or to override
the defaults.

This shortens the distance between a service specific configuration file (such
as `cinder.conf`) and what the human operator provides in the manifests.

These configuration snippets are passed to the operators in the different
`customServiceConfig` sections available in the manifests, and then they are
layered in the services available in the following levels. To illustrate this,
if we were to set a configuration at the top Cinder level (`spec: cinder:
template:`) then it would be applied to all the cinder services; for example to
enable debug in all the cinder services we would do:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
spec:
  cinder:
    template:
      customServiceConfig: |
        [DEFAULT]
        debug = True
< . . . >
```

If we only wanted to set it for one of the cinder services, for example the
scheduler, then we would use the `cinderScheduler` section instead:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
spec:
  cinder:
    template:
      cinderScheduler:
        customServiceConfig: |
          [DEFAULT]
          debug = True
< . . . >
```

In openshift it is not recommended to store sensitive information like the
credentials to the cinder storage array in the CRs, so most OpenStack operators
have a mechanism to use OpenShift's `Secrets` for sensitive configuration
parameters of the services and then use then by reference in the
`customServiceConfigSecrets` section which is analogous to the
`customServiceConfig`.

The contents of the `Secret` references passed in the
`customServiceConfigSecrets` will have the same format as `customServiceConfig`:
a snippet with the section/s and configuration options.

When there are sensitive information in the service configuration then it
becomes a matter of personal preference whether to store all the configuration
in the `Secret` or only the sensitive parts, but remember that if we split the
configuration between `Secret` and `customServiceConfig` we still need the
section header (eg: `[DEFAULT]`) to be present in both places.

Attention should be paid to each service's adoption process as they may have
some particularities regarding their configuration.

## Configuration tooling

In order to help users to handle the configuration for the TripleO and Openstack
services the tool: https://github.com/openstack-k8s-operators/os-diff has been
develop to compare the configuration files between the TripleO deployment and
the next gen cloud.
Make sure Golang is installed and configured on your env:

```bash
git clone https://github.com/openstack-k8s-operators/os-diff
pushd os-diff
make build
```

Then configure ansible.cfg and ssh-config file according to your environment:

```yaml
Host *
    IdentitiesOnly yes

Host virthost
    Hostname virthost
    IdentityFile ~/.ssh/id_rsa
    User root
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null


Host standalone
    Hostname standalone
    IdentityFile ~/install_yamls/out/edpm/ansibleee-ssh-key-id_rsa
    User root
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null

Host crc
    Hostname crc
    IdentityFile ~/.ssh/id_rsa
    User stack
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
```

And test your connection:

```bash
ssh -F ssh.config standalone
```

## Node Roles

In Director deployments we had 4 different standard roles for the nodes:
`Controller`, `Compute`, `Ceph Storage`, `Swift Storage`, but in podified
OpenStack we just make a distinction based on where things are running, in
OpenShift or external to it.

When adopting a Director OpenStack your `Compute` nodes will directly become
external nodes, so there should not be much additional planning needed there.

In many deployments being adopted the `Controller` nodes will require some
thought because we'll have many OpenShift nodes where the controller services
could run, and we have to decide which ones we want to use, how we are going to
use them, and make sure those nodes are ready to run the services.

In most deployments running OpenStack services on `master` nodes can have a
seriously adverse impact on the OpenShift cluster, so we recommend placing
OpenStack services on non `master` nodes.

By default OpenStack Operators deploy OpenStack services on any worker node, but
that is not necessarily what's best for all deployments, and there may be even
services that won't even work deployed like that.

When planing a deployment it's good to remember that not all the services on an
OpenStack deployments are the same as they have very different requirements.

Looking at the Cinder component we can clearly see different requirements for
its services: the cinder-scheduler is a very light service with low
memory, disk, network, and CPU usage; cinder-api service has a higher network
usage due to resource listing requests; the cinder-volume service will have a
high disk and network usage since many of its operations are in the data path
(offline volume migration, create volume from image, etc.), and then we have
the cinder-backup service which has high memory, network, and CPU (to compress
data) requirements.

We also have the Glance and Swift components that are in the data path, and
let's not forget RabbitMQ and Galera services.

Given these requirements it may be preferable not to let these services wander
all over your OpenShift worker nodes with the possibility of impacting other
workloads, or maybe you don't mind the light services wandering around but you
want to pin down the heavy ones to a set of infrastructure nodes.

There are also hardware restrictions to take into consideration, because if we
are using a Fibre Channel (FC) Cinder backend we'll need the cinder-volume,
cinder-backup, and maybe even the glance (if it's using Cinder as a backend)
services to run on a OpenShift host that has an HBA.

The OpenStack Operators allow a great deal of flexibility on where to run the
OpenStack services, as we can use node labels to define which OpenShift nodes
are eligible to run the different OpenStack services.  Refer to the [Node
Selector guide](node-selector.md) to learn more about using labels to define
placement of the OpenStack services.

**TODO: Talk about Ceph Storage and Swift Storage nodes, HCI deployments,
etc.**

## Network

**TODO: Write about isolated networks, NetworkAttachmentDefinition,
NetworkAttachmets, etc**

## Storage

When looking into the storage in an OpenStack deployment we can differentiate
2 different kinds, the storage requirements of the services themselves and the
storage used for the OpenStack users that thee services will manage.

These requirements may drive our OpenShift node selection, as mentioned above,
and may even require us to do some preparations on the OpenShift nodes before
we can deploy the services.

**TODO: Galera, RabbitMQ, Swift, Glance, etc.**

### Cinder requirements

The Cinder service has both local storage used by the service and OpenStack user
requirements.

Local storage is used for example when downloading a glance image for the create
volume from image operation, which can become considerable when having
concurrent operations and not using cinder volume cache.

In the Operator deployed OpenStack we now have an easy way to configure the
location of the conversion directory to be an NFS share (using the extra
volumes feature), something that needed to be done manually before.

Even if it's an adoption and it may seem that there's nothing to consider
regarding the Cinder backends, because we'll just be using the same ones we are
using in our current deployment, we should still evaluate it, because it may not
be so straightforward.

First we need to check the transport protocol the Cinder backends are using:
RBD, iSCSI, FC, NFS, NVMe-oF, etc.

Once we know all the transport protocols we are using, we can proceed to make
sure we are taking them into consideration when placing the Cinder services
(as mentioned above in the Node Roles section) and the right storage transport
related binaries are running on the OpenShift nodes.

Detailed information about the specifics for each storage transport protocol can
be found in the [Cinder Adoption section](cinder_adoption.md).  Please take a
good look at that document before proceeding to be able to plan the adoption
better.
