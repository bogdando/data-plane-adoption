# Pull Openstack configuration

Before starting to adoption workflow, we can start by pulling the configuration
from the Openstack services and TripleO on our file system in order to backup
the configuration files and then use it for later, during the configuration of
the adopted services and for the record to compare and make sure nothing has been
missed or misconfigured.

Make sure you have pull the os-diff repository and configure according to your
environment:
[Configure os-diff](planning.md#Configuration tooling)

## Pull configuration from a TripleO deployment

Once you make sure the ssh connnection is confugred correctly and os-diff has been
built, you can start to pull configuration from your Openstack services.

All the services are describes in an Ansible role:

[collect_config vars](https://github.com/openstack-k8s-operators/os-diff/blob/main/roles/collect_config/vars/main.yml)

Once you enabled the services you need (you can enable everything even if a services is not deployed)
you can start to pull the Openstack services configuration files:

```bash
pushd os-diff
./os-diff pull --cloud_engine=podman
```

The configuration will be pulled and stored in:
```bash
/tmp/collect_tripleo_configs
```

And you provided another path with:

```bash
./os-diff pull --cloud_engine=podman -e local_working_dir=$HOME
```

Once the ansible playbook has been run, you should have into your local directory a directory per services

```
  ▾ tmp/
    ▾ collect_tripleo_configs/
      ▾ glance/
```
