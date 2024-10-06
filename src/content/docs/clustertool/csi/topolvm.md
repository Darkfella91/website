TopoLVM: A CSI Plugin for Kubernetes
Work In Progress

This program, along with all its features and general design, is a work in progress. It is currently not finished and not widely available. All code and documentation are considered pre-beta drafts.

TopoLVM is a Container Storage Interface (CSI) plugin that utilizes Logical Volume Manager (LVM) for Kubernetes. It serves as a specific implementation of local persistent volumes using CSI and LVM.

You can find the source code and Helm chart for TopoLVM on its GitHub repository.

Note: Nothing in this guide is specific to TrueCharts. While there are some Talos-specific steps, these can be performed more simply on other operating systems.

Requirements
To provision storage, TopoLVM requires its own LVM Volume Group. In this guide, we will assume you are providing a separate drive (or virtual disk) specifically for TopoLVM to simplify the setup.

LVM Preparation
Identify the disk you want to use for TopoLVM. If you are using Talos OS, you can list the available disks with the following command:

talosctl disks

Ensure you have installed an additional disk on your VM or bare-metal server if necessary.

Use TrueCharts' LVM_disk_watcher chart and container to configure the disk.

TODO: Add LVM_DISK_WATCHER configuration.

Install TopoLVM
Now that the node is prepared to create volumes for TopoLVM, follow these steps to install it:

Add the TopoLVM repository to your cluster using the Helm CLI:
   helm repo add topolvm https://topolvm.github.io/topolvm/

Install the Helm chart with the following command:
   helm install topolvm topolvm/topolvm --version 15.4.0 -f path/to/your/values.yaml -n topolvm-system

Helm Values Configuration
When configuring Helm values, pay special attention to the following settings:

Add a device class that will use your thin pool.
Define a storage class that will utilize that device class.
For more detailed information on how TopoLVM works, refer to the documentation in the GitHub repository.

values:
  # LVMD Service Configuration
  lvmd:
    managed: false
    env:
      - name: LVM_SYSTEM_DIR
        value: /tmp
    deviceClasses:
      - name: main-thin
        volume-group: topolvm_vg
        default: true
        spare-gb: 10
        type: thin
        thin-pool:
          name: topolvm_thin
          overprovision-ratio: 10.0
  node:
    lvmdEmbedded: true

  # Storage Classes Configuration
  storageClasses:
    - name: topolvm-thin-provisioner
      storageClass:
        fsType: xfs  # Supported filesystems: ext4, xfs, btrfs.
        reclaimPolicy: Delete
        annotations: {}
        isDefaultClass: true
        volumeBindingMode: WaitForFirstConsumer  # Recommended for optimal scheduling.
        allowVolumeExpansion: true
        additionalParameters:
          "topolvm.io/device-class": "main-thin"  # Matches the device class specified above.
        mountOptions: []

Snapshots
To be determined.

Optional: Non-ClusterTool Only
The following steps are included in ClusterTool by default.

Kernel Modules
To ensure proper functionality, add the following kernel modules. You can use modprobe for typical Linux installations or add them to your talconfig.yaml if using TalHelper or ClusterTool, as shown below:

yaml
Copy code
# talconfig.yaml
nodes:
  - hostname: k8s-control-1
    kernelModules:
      - name: dm_thin_pool
      - name: dm_mod

Manually Add Disks
Data Loss

These steps could lead to data loss if performed on the wrong disks.

The following commands set up a Volume Group and Thin Pool for TopoLVM. Ensure that the names match your setup.

Create a Physical Volume:

bash
Copy code
pvcreate /dev/vdb
Create a Volume Group:

bash
Copy code
vgcreate topolvm_vg /dev/vdb
Create a Thin Pool:

bash
Copy code
lvcreate -l 100%FREE --chunksize 256 -T -A n -n topolvm_thin topolvm_vg
Create Privileged Namespace
Create the namespace with the following labels:

yaml
Copy code
apiVersion: v1
kind: Namespace
metadata:
  name: topolvm-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    topolvm.io/webhook: ignore

Additional References
For more guidance, consult the following resources: Thin Volumes Proposal.
