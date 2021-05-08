# nfs-volume-provisioner
An NFS volume provisioner for Kubernetes

This provisioner is based on the [Ganesha NFS Server Provisioner](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner)
which is an out-of-tree dynamic provisioner for Kubernetes. You can use it to quickly & easily deploy shared storage that works almost anywhere.

This chart will deploy the Kubernetes `nfs` provisioner. This provisioner includes a built in NFS server, and is not intended for connecting to a pre-existing
NFS server. If you have a pre-existing NFS Server, consider using the Ganesha nfs client provisioner instead.

## Introduction
This chart has been modified from the default Ganesha provisioner to work on digital ocean. It has also been optimized for the ndtech use cases.

## Installing the Chart
ndtech uses flux to install this chart but it should also be compatible with a command line helm install process.

## Uninstalling the Chart
Remove the helm release from your flux repo

## Configuration

The following table lists the configurable parameters of the kibana chart and
their default values.

| Parameter                      | Description                                                                                                     | Default                                                  |
|:-------------------------------|:----------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------|
| `extraArgs` | [Additional command line arguments](https://github.com/kubernetes-incubator/external-storage/blob/master/nfs/docs/deployment.md#arguments) | `{}`
| `imagePullSecrets`             | Specify image pull secrets                                                                                      | `nil` (does not add image pull secrets to deployed pods) |
| `image.repository`             | The image repository to pull from                                                                               | `gcr.io/k8s-staging-sig-storage/nfs-provisioner`         |
| `image.tag`                    | The image tag to pull                                                                                           | `v3.0.0`                                                 |
| `image.digest`                 | The image digest to pull, this option has precedence over `image.tag`                                           | `nil`                                                    |
| `image.pullPolicy`             | Image pull policy                                                                                               | `IfNotPresent`                                           |
| `service.type`                 | service type                                                                                                    | `ClusterIP`                                              |
| `service.nfsPort`              | TCP port on which the nfs-server-provisioner NFS service is exposed                                                    | `2049`                                                   |
| `service.mountdPort`           | TCP port on which the nfs-server-provisioner mountd service is exposed                                                 | `20048`                                                  |
| `service.rpcbindPort`          | TCP port on which the nfs-server-provisioner RPC service is exposed                                                    | `111`                                                    |
| `service.nfsNodePort`          | if `service.type` is `NodePort` and this is non-empty, sets the nfs-server-provisioner node port of the NFS service    | `nil`                                                    |
| `service.mountdNodePort`       | if `service.type` is `NodePort` and this is non-empty, sets the nfs-server-provisioner node port of the mountd service | `nil`                                                    |
| `service.rpcbindNodePort`      | if `service.type` is `NodePort` and this is non-empty, sets the nfs-server-provisioner node port of the RPC service    | `nil`                                                    |
| `persistence.enabled`          | Enable config persistence using PVC                                                                             | `false`                                                  |
| `persistence.storageClass`     | PVC Storage Class for config volume                                                                             | `nil`                                                    |
| `persistence.accessMode`       | PVC Access Mode for config volume                                                                               | `ReadWriteOnce`                                          |
| `persistence.size`             | PVC Storage Request for config volume                                                                           | `1Gi`                                                    |
| `storageClass.create`          | Enable creation of a StorageClass to consume this nfs-server-provisioner instance                                      | `true`                                                   |
| `storageClass.provisionerName` | The provisioner name for the storageclass                                                                       | `cluster.local/{release-name}-{chart-name}`              |
| `storageClass.defaultClass`    | Whether to set the created StorageClass as the clusters default StorageClass                                    | `false`                                                  |
| `storageClass.name`            | The name to assign the created StorageClass                                                                     | `nfs`                                                    |
| `storageClass.allowVolumeExpansion` | Allow base storage PCV to be dynamically resizeable (set to null to disable )                              | `true                                                    |
| `storageClass.parameters`      | Parameters for StorageClass                                                                                     | `{}`                                                     |
| `storageClass.mountOptions`    | Mount options for StorageClass                                                                                  | `[ "vers=3" ]`                                           |
| `storageClass.reclaimPolicy`   | ReclaimPolicy field of the class, which can be either Delete or Retain                                          | `Delete`                                                    |
| `resources`                    | Resource limits for nfs-server-provisioner pod                                                                          | `{}`                                                     |
| `nodeSelector`                 | Map of node labels for pod assignment                                                                           | `{}`                                                     |
| `tolerations`                  | List of node taints to tolerate                                                                                 | `[]`                                                     |
| `affinity`                     | Map of node/pod affinities                                                                                      | `{}`                                                     |
| `podSecurityContext`              | Security context settings for nfs-server-provisioner pod (see https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod)                                                        | `{}`                                                     |

```console
$ helm install stable/nfs-server-provisioner --name my-release \
  --set=image.tag=v1.0.8,resources.limits.cpu=200m
```

Alternatively, a YAML file that specifies the values for the above parameters
can be provided while installing the chart. For example,

```console
$ helm install stable/nfs-server-provisioner --name my-release -f values.yaml
```

> **Tip**: You can use the default [values.yaml](values.yaml) as an example

## Persistence

The nfs-server-provisioner image stores it's configuration data, and importantly, **the dynamic volumes it
manages** `/export` path of the container.

The chart mounts a [Persistent Volume](http://kubernetes.io/docs/user-guide/persistent-volumes/)
volume at this location. The volume can be created using dynamic volume
provisioning. However, **it is highly recommended** to explicitly specify
a storageclass to use rather than accept the clusters default, or pre-create
a volume for each replica.

If this chart is deployed with more than 1 replica, `storageClass.defaultClass=true`
and `persistence.storageClass`, then the 2nd+ replica will end up using the 1st
replica to provision storage - which is likely never a desired outcome.

## Recommended Persistence Configuration Examples

The following is a recommended configuration example when another storage class
exists to provide persistence:

    persistence:
      enabled: true
      storageClass: "standard"
      size: 200Gi

    storageClass:
      defaultClass: true

On many clusters, the cloud provider integration will create a "standard" storage
class which will create a volume (e.g. a Google Compute Engine Persistent Disk or
Amazon EBS volume) to provide persistence.

---

The following is a recommended configuration example when another storage class
does not exist to provide persistence:

    persistence:
      enabled: true
      storageClass: "-"
      size: 200Gi

    storageClass:
      defaultClass: true

In this configuration, a `PersistentVolume` must be created for each replica
to use. Installing the Helm chart, and then inspecting the `PersistentVolumeClaim`'s
created will provide the necessary names for your `PersistentVolume`'s to bind to.

An example of the necessary `PersistentVolume`:

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: data-nfs-server-provisioner-0
    spec:
      capacity:
        storage: 200Gi
      accessModes:
        - ReadWriteOnce
      gcePersistentDisk:
        fsType: "ext4"
        pdName: "data-nfs-server-provisioner-0"
      claimRef:
        namespace: kube-system
        name: data-nfs-server-provisioner-0

---

The following is a recommended configration example for running on bare metal with a hostPath volume:

    persistence:
      enabled: true
      storageClass: "-"
      size: 200Gi

    storageClass:
      defaultClass: true

    nodeSelector:
      kubernetes.io/hostname: {node-name}

In this configuration, a `PersistentVolume` must be created for each replica
to use. Installing the Helm chart, and then inspecting the `PersistentVolumeClaim`'s
created will provide the necessary names for your `PersistentVolume`'s to bind to.

An example of the necessary `PersistentVolume`:

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: data-nfs-server-provisioner-0
    spec:
      capacity:
        storage: 200Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /srv/volumes/data-nfs-server-provisioner-0
      claimRef:
        namespace: kube-system
        name: data-nfs-server-provisioner-0

> **Warning**: `hostPath` volumes cannot be migrated between machines by Kubernetes, as such,
in this example, we have restricted the `nfs-server-provisioner` pod to run on a single node. This
is unsuitable for production deployments.
