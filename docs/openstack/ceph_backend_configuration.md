# Ceph backend configuration (if applicable)

If the original deployment uses a Ceph storage backend for any service
(e.g. Glance, Cinder, Nova), the same backend must be used in the
adopted deployment and CRs must be configured accordingly.

## Prerequisites

* The `OpenStackControlPlane` CR must already exist.

## Variables

Define the shell variables used in the steps below. The values are
just illustrative, use values that are correct for your environment:

```
CEPH_SSH="ssh -i ~/install_yamls/out/edpm/ansibleee-ssh-key-id_rsa root@192.168.122.100"
CEPH_KEY=$($CEPH_SSH "cat /etc/ceph/ceph.client.openstack.keyring | base64 -w 0")
CEPH_CONF=$($CEPH_SSH "cat /etc/ceph/ceph.conf | base64 -w 0")
```

## Procedure - Ceph backend configuration

Create the `ceph-conf-files` secret, containing Ceph configuration:

```
oc apply -f - <<EOF
apiVersion: v1
data:
  ceph.client.openstack.keyring: $CEPH_KEY
  ceph.conf: $CEPH_CONF
kind: Secret
metadata:
  name: ceph-conf-files
  namespace: openstack
type: Opaque
EOF
```

The content of the file should look something like this:

> ```
> ---
> apiVersion: v1
> kind: Secret
> metadata:
>   name: ceph-conf-files
>   namespace: openstack
> stringData:
>   ceph.client.openstack.keyring: |
>     [client.openstack]
>         key = <secret key>
>         caps mgr = "allow *"
>         caps mon = "profile rbd"
>         caps osd = "profile rbd pool=images"
>   ceph.conf: |
>     [global]
>     fsid = 7a1719e8-9c59-49e2-ae2b-d7eb08c695d4
>     mon_host = 10.1.1.2,10.1.1.3,10.1.1.4
> ```

Configure `extraMounts` within the `OpenStackControlPlane` CR:

```
oc patch openstackcontrolplane openstack --type=merge --patch '
spec:
  extraMounts:
    - name: v1
      region: r1
      extraVol:
        - propagation:
          - CinderVolume
          - CinderBackup
          - GlanceAPI
          extraVolType: Ceph
          volumes:
          - name: ceph
            projected:
              sources:
              - secret:
                  name: ceph-conf-files
          mounts:
          - name: ceph
            mountPath: "/etc/ceph"
            readOnly: true
'
```

## Getting Ceph FSID

Configuring some OpenStack services to use Ceph backend may require
the FSID value. You can fetch the value from the config like so:

```
CEPH_FSID=$(oc get secret ceph-conf-files -o json | jq -r '.data."ceph.conf"' | base64 -d | grep fsid | sed -e 's/fsid = //')
```
