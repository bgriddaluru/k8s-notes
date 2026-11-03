

Controller component - (Deployment/ReplicaSet)
1. Among the controller sidecars are an external-provisioner, an external-attacher i.e. creating/deleting and attaching/detaching volumes.
   1. external-provisioner: Sidecar container that watches Kubernetes PersistentVolumeClaim objects and triggers CreateVolume/DeleteVolume against a CSI endpoint to create a PersistentVolumeObject
   2. external-attacher:  Sidecar container that watches Kubernetes VolumeAttachment objects and triggers ControllerPublish/Unpublish against a CSI endpoint
2. ControllerPublishVolume: This method is used make a volume available on some required node

Per-node Component - (DaemonSet)
1. The node component communicates with the kubelet, which runs across all nodes and handles calls for the CSI Node service. 
2. The calls can mount or unmount storage volumes from a storage system and make them available for the pod to consume i.e. formats and mounts the volume that is attached to a given volume.
3. NodeStageVolume: This method is called by the CO to temporarily mount the volume to a staging path. Usually this staging path is a global directory on the node. In Kubernetes, after it's mounted to the global directory, you mount it into the pod directory (via NodePublishVolume). The reason that mounting is a two step operation is because Kubernetes allows you to use a single volume by multiple pods. This is allowed when the storage system supports it (say NFS) or if all pods run on the same node. One thing to note is that you also need to format  the volume if it's not formatted already. Keep that in mind.
4. NodePublishVolume: This method is called to mount the volume from staging to target path. Usually what you do here is a bind mount. A bind mount allows you to mount a path to a different path

driver-registrar:  Sidecar container that 1) registers the CSI driver with kubelet, and 2) adds  the drivers custom NodeId to a label on the Kubernetes Node API Object.






### Documents

Original Design: https://github.com/kubernetes/design-proposals-archive/blob/main/storage/container-storage-interface.md
Arslan  Notes: https://arslan.io/2018/06/21/how-to-write-a-container-storage-interface-csi-plugin/