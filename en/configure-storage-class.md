---
title: Persistent Storage Class Configuration on Kubernetes
summary: Learn how to configure local PVs and network PVs.
aliases: ['/docs/tidb-in-kubernetes/dev/configure-storage-class/','/docs/dev/tidb-in-kubernetes/reference/configuration/local-pv/']
---

# Persistent Storage Class Configuration on Kubernetes

TiDB cluster components such as PD, TiKV, TiDB monitoring, TiDB Binlog, and `tidb-backup` require the persistent storage of data. To persist the data on Kubernetes, you need to use [PersistentVolume (PV)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). Kubernetes supports several types of [storage classes](https://kubernetes.io/docs/concepts/storage/volumes/), which are mainly divided into two parts:

- Network storage

    The network storage medium is not on the current node but is mounted to the node through the network. Generally, there are redundant replicas to guarantee high availability. When the node fails, the corresponding network storage can be re-mounted to another node for further use.

- Local storage

    The local storage medium is on the current node and typically can provide lower latency than the network storage. Because there are no redundant replicas, once the node fails, data might be lost. If it is an IDC server, data can be restored to a certain extent. If it is a virtual machine using the local disk on the public cloud, data **cannot** be retrieved after the node fails.

PVs are created automatically by the system administrator or volume provisioner. PVs and Pods are bound by [PersistentVolumeClaim (PVC)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims). Users request for using a PV through a PVC instead of creating a PV directly. The corresponding volume provisioner creates a PV that meets the requirements of PVC and then binds the PV to the PVC.

> **Warning:**
>
> Do not delete a PV in any case unless you are familiar with the underlying volume provisioner. Deleting a PV manually can cause orphaned volumes and unexpected behavior.

## Recommended storage classes for TiDB clusters

TiKV uses the Raft protocol to replicate data. When a node fails, PD automatically schedules data to fill the missing data replicas; TiKV requires low read and write latency, so local SSD storage is strongly recommended in the production environment.

PD also uses Raft to replicate data. PD is not an I/O-intensive application, but a database for storing cluster meta information, so a local SAS disk or network SSD storage such as EBS General Purpose SSD (gp2) volumes on AWS or SSD persistent disks on GCP can meet the requirements.

To ensure availability, it is recommended to use network storage for components such as TiDB monitoring, TiDB Binlog, and `tidb-backup` because they do not have redundant replicas. TiDB Binlog's Pump and Drainer components are I/O-intensive applications that require low read and write latency, so it is recommended to use high-performance network storage such as EBS Provisioned IOPS SSD (io1) volumes on AWS or SSD persistent disks on GCP.

When deploying TiDB clusters or `tidb-backup` with TiDB Operator, you can configure `StorageClass` for the components that require persistent storage via the corresponding `storageClassName` field in the `values.yaml` configuration file. The `StorageClassName` is set to `local-storage` by default.

## Network PV configuration

Kubernetes 1.11 and later versions support [volume expansion of network PV](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/), but you need to run the following command to enable volume expansion for the corresponding `StorageClass`:

{{< copyable "shell-regular" >}}

```shell
kubectl patch storageclass ${storage_class} -p '{"allowVolumeExpansion": true}'
```

After volume expansion is enabled, expand the PV using the following method:

1. Edit the PersistentVolumeClaim (PVC) object:

    Suppose the PVC is 10 Gi and now we need to expand it to 100 Gi.

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl patch pvc -n ${namespace} ${pvc_name} -p '{"spec": {"resources": {"requests": {"storage": "100Gi"}}}}'
    ```

2. View the size of the PV:

    After the expansion, the size displayed by running `kubectl get pvc -n ${namespace} ${pvc_name}` is still the original one. But if you run the following command to view the size of the PV, it shows that the size has been expanded to the expected one.

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl get pv | grep ${pvc_name}
    ```

## Local PV configuration

Kubernetes currently supports statically allocated local storage. To create a local storage object, use `local-volume-provisioner` in the [local-static-provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) repository.

### Step 1: Pre-allocate local storage

- For a disk that stores TiKV data, you can [mount](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/operations.md#use-a-whole-disk-as-a-filesystem-pv) the disk into the `/mnt/ssd` directory.

    To achieve high performance, it is recommanded to allocate TiDB a dedicated disk, and the recommended disk type is SSD.

- For a disk that stores PD data, follow the [steps](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/operations.md#sharing-a-disk-filesystem-by-multiple-filesystem-pvs) to mount the disk. First, create multiple directories in the disk, and bind mount the directories into the `/mnt/sharedssd` directory.

    >**Note:**
    >
    > The number of directories you create depends on the planned number of TiDB clusters, and the number of PD servers in each cluster. For each directory, a corresponding PV will be created. Each PD server uses one PV.

- For a disk that stores monitoring data, follow the [steps](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/operations.md#sharing-a-disk-filesystem-by-multiple-filesystem-pvs) to mount the disk. First, create multiple directories in the disk, and bind mount the directories into the `/mnt/monitoring` directory.

    >**Note:**
    >
    > The number of directories you create depends on the planned number of TiDB clusters. For each directory, a corresponding PV will be created. The monitoring data in each TiDB cluster uses one PV.

- For a disk that stores TiDB Binlog and backup data, follow the [steps](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/operations.md#sharing-a-disk-filesystem-by-multiple-filesystem-pvs) to mount the disk. First, create multiple directories in the disk, and bind mount the directories them into the `/mnt/backup` directory.

    >**Note:**
    >
    > The number of directories you create depends on the planned number of TiDB clusters, the number of Pumps in each cluster, and your backup method. For each directory, a corresponding PV will be created. Each Pump uses one PV and each Drainer uses one PV. All [Ad-hoc full backup](backup-to-s3.md#ad-hoc-full-backup-to-s3-compatible-storage) tasks and all [scheduled full backup](backup-to-s3.md#scheduled-full-backup-to-s3-compatible-storage) tasks share one PV.

The `/mnt/ssd`, `/mnt/sharedssd`, `/mnt/monitoring`, and `/mnt/backup` directories mentioned above are discovery directories used by local-volume-provisioner. local-volume-provisioner creates a corresponding PV for each subdirectory in discovery directory.

### Step 2: Deploy local-volume-provisioner

#### Online deployment

1. Download the deployment file for local-volume-provisioner.

    {{< copyable "shell-regular" >}}

    ```shell
    wget https://raw.githubusercontent.com/pingcap/tidb-operator/master/examples/local-pv/local-volume-provisioner.yaml
    ```

2. If you use the same discovery directory as described in [Step 1: Pre-allocate local storage](#step-1-pre-allocate-local-storage), you can skip this step. If you use a different path of discovery directory than in the previous step, you need to modify the ConfigMap and DaemonSet spec.

    * Modify the `data.storageClassMap` field in the ConfigMap spec:

        ```yaml
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: local-provisioner-config
          namespace: kube-system
        data:
          # ...
          storageClassMap: |
            ssd-storage:
              hostDir: /mnt/ssd
              mountDir: /mnt/ssd
            shared-ssd-storage:
              hostDir: /mnt/sharedssd
              mountDir: /mnt/sharedssd
            monitoring-storage:
              hostDir: /mnt/monitoring
              mountDir: /mnt/monitoring
            backup-storage:
              hostDir: /mnt/backup
              mountDir: /mnt/backup
        ```

        For more configuration about local-volume-provisioner, refer to [Configuration](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/provisioner.md#configuration).

    * Modify `volumes` and `volumeMounts` fields in the DaemonSet spec to ensure the discovery directory can be mounted to the corresponding directory in the Pod:

        ```yaml
        ......
              volumeMounts:
                - mountPath: /mnt/ssd
                  name: local-ssd
                  mountPropagation: "HostToContainer"
                - mountPath: /mnt/sharedssd
                  name: local-sharedssd
                  mountPropagation: "HostToContainer"
                - mountPath: /mnt/backup
                  name: local-backup
                  mountPropagation: "HostToContainer"
                - mountPath: /mnt/monitoring
                  name: local-monitoring
                  mountPropagation: "HostToContainer"
          volumes:
            - name: local-ssd
              hostPath:
                path: /mnt/ssd
            - name: local-sharedssd
              hostPath:
                path: /mnt/sharedssd
            - name: local-backup
              hostPath:
                path: /mnt/backup
            - name: local-monitoring
              hostPath:
                path: /mnt/monitoring
        ......
        ```

3. Deploy `local-volume-provisioner`.

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/manifests/local-dind/local-volume-provisioner.yaml
    ```

4. Check status of Pod and PV.

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl get po -n kube-system -l app=local-volume-provisioner && \
    kubectl get pv | grep -e ssd-storage -e shared-ssd-storage -e monitoring-storage -e backup-storage
    ```

    `local-volume-provisioner` creates a PV for each mounting point under the discovery directory.

    > **Note:**
    >
    > If no mount point is in the discovery directory, no PV is created and the output is empty.

For more information, refer to [Kubernetes local storage](https://kubernetes.io/docs/concepts/storage/volumes/#local) and [local-static-provisioner document](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner#overview).

#### Offline deployment

Steps of offline deployment is same as online deployment, except the following:

* Download the `local-volume-provisioner.yaml` file on a machine with Internet access, then upload it to the server and install it.

* `local-volume-provisioner` is a DaemonSet that starts a Pod on every Kubernetes worker node. The Pod uses the `quay.io/external_storage/local-volume-provisioner:v2.3.4` image. If the server does not have access to the Internet, download this Docker image on a machine with Internet access:

    {{< copyable "shell-regular" >}}

    ``` shell
    docker pull quay.io/external_storage/local-volume-provisioner:v2.3.4
    docker save -o local-volume-provisioner-v2.3.4.tar quay.io/external_storage/local-volume-provisioner:v2.3.4
    ```

    Copy the `local-volume-provisioner-v2.3.4.tar` file to the server, and execute the `docker load` command to load the file on the server:

    {{< copyable "shell-regular" >}}

    ```shell
    docker load -i local-volume-provisioner-v2.3.4.tar
    ```

### Best practices

- A local PV's path is its unique identifier. To avoid conflicts, it is recommended to use the UUID of the device to generate a unique path.
- For I/O isolation, a dedicated physical disk per PV is recommended to ensure hardware-based isolation.
- For capacity isolation, a partition per PV or a physical disk per PV is recommended.

For more information on local PV on Kubernetes, refer to [Best Practices](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/best-practices.md).

## Data safety

In general, after a PVC is no longer used and deleted, the PV bound to it is reclaimed and placed in the resource pool for scheduling by the provisioner. To avoid accidental data loss, you can globally configure the reclaim policy of the `StorageClass` to `Retain` or only change the reclaim policy of a single PV to `Retain`. With the `Retain` policy, a PV is not automatically reclaimed.

- Configure globally:

    The reclaim policy of a `StorageClass` is set at creation time and it cannot be updated once it is created. If it is not set when created, you can create another `StorageClass` of the same provisioner. For example, the default reclaim policy of the `StorageClass` for persistent disks on Google Kubernetes Engine (GKE) is `Delete`. You can create another `StorageClass` named `pd-standard` with its reclaim policy as `Retain`, and change the `storageClassName` of the corresponding component to `pd-standard` when creating a TiDB cluster.

    {{< copyable "" >}}

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: pd-standard
    parameters:
       type: pd-standard
    provisioner: kubernetes.io/gce-pd
    reclaimPolicy: Retain
    volumeBindingMode: Immediate
    ```

- Configure a single PV:

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl patch pv ${pv_name} -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
    ```

> **Note:**
>
> By default, to ensure data safety, TiDB Operator automatically changes the reclaim policy of the PVs of PD and TiKV to `Retain`.

### Delete PV and data

When the reclaim policy of PVs is set to `Retain`, if you have confirmed that the data of a PV can be deleted, you can delete this PV and the corresponding data by strictly taking the following steps:

1. Delete the PVC object corresponding to the PV:

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl delete pvc ${pvc_name} --namespace=${namespace}
    ```

2. Set the reclaim policy of the PV to `Delete`. Then the PV is automatically deleted and reclaimed.

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl patch pv ${pv_name} -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
    ```

For more details, refer to [Change the Reclaim Policy of a PersistentVolume](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/).
