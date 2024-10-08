=======================================
Lab 02 - Understand Kubernetes Storage
=======================================


In this step, we will create a StorageClass for Portworx volumes.

Understand StorageClass
-----------------------

A `StorageClass <https://kubernetes.io/docs/concepts/storage/storage-classes/>`__ provides a way for administrators to describe the “classes” of storage on their Kubernetes cluster. Before we create a volume in Kubernetes, we have to create a StorageClass.

For example, the following is a Portworx StorageClass whose volumes have replication factor of 3 and high IO priority. \* Replication factor of 3 means the data for the volume is replicated on 3 different nodes in the cluster \* High IO priority means Portworx will use storage devices that are classified into the high IO profile (for e.g SSDs).

Create the storage class by running

.. code-block:: shell

  cat <<EOF | oc create -f -
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-repl3-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    priority_io: "high"
  reclaimPolicy: Delete
  volumeBindingMode: Immediate
  EOF


Let's proceed to creating volumes that use this storage class.

In this step, we will deploy a ``PersistentVolumeClaim`` using Portworx.

Understand PersistentVolumeClaim
--------------------------------------

A `PersistentVolumeClaim <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims>`__ can be used to dynamically create a volume using Portworx.

For example, below is the spec for a 2GB volume that uses the Portworx Storage class we created before this step.

Let's create the PersistentVolumeClaim:

.. code-block:: shell

  cat <<EOF | oc create -f -
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: px-pvc
  spec:
    storageClassName: px-repl3-sc
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 2Gi
  EOF


Behind the scenes, Kubernetes talks to the Portworx native driver to create this PVC. Each PVC has a unique one-one mapping to a `PersistentVolume <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>`__ which is the actual volume backing the PVC.

Validate PersistentVolumeClaim
------------------------------------

A PersistentVolumeClaim is successfully provisioned once it gets into “Bound” state. Let's run the below script to check that.

.. code-block:: shell

  oc get pvc

we can see in the example output that our PVC is bound.

Let's proceed to the next step to further inspect the volume.

In this step, we will use ``pxctl`` to inspect the volume.

Inspect the Portworx volume
---------------------------

Portworx ships with a `pxctl <https://docs.portworx.com/portworx-enterprise/reference/cli/pxctl-reference/status-reference>`__ command line that can be used to manage Portworx.

Below we will use pxctl to inspect the underlying volume for our PVC.

.. code-block:: shell

  VOL=$(oc get pvc | grep px-pvc | awk '{print $3}')
  pxctl volume inspect ${VOL}

Make the following observations in the inspect output \* ``HA`` shows the number of configured replcas for this volume \* ``Labels`` show the name of the PVC for this volume \* ``Replica sets on nodes`` shows the px nodes on which volume is replicated \* ``State`` indicates the volume is detached which means no applications are using the volume yet
