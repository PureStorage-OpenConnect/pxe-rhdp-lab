===========================
Lab 06 - Snapshot Schedules
===========================


Create a new snapshot schedule policy
-------------------------------------

Create a daily snapshot schedule policy called ``daily-schedule`` at ``10 PM``, ``retain 5``.

.. code-block:: shell

  cat <<EOF | oc apply -f -
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: SchedulePolicy
  metadata:
    name: daily-schedule
  policy:
    daily:
      time: "10:00PM"
      retain: 5
  EOF



Create a storageClass that uses this schedule policy
----------------------------------------------------

Create a storage class ``px-nginx-scheduled`` with the newly created schedule policy ``daily-schedule``

.. code-block:: shell

  cat <<EOF | oc apply -f -
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-nginx-scheduled
  provisioner: pxd.portworx.com
  parameters:
    repl: "2"
    io_priority: "high"
    snapshotschedule.stork.libopenstorage.org/default-schedule: |
      schedulePolicyName: default-interval-schedule
      annotations:
        portworx/snapshot-type: local
  EOF



Create a Nginx StatefulSet that utilizes this storageClass
----------------------------------------------------------

Create a new NGINX StatefulSet, making use of the ``px-nginx-scheduled`` storage class.



.. code-block:: shell

  cat <<EOF | oc apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx
    labels:
      app: nginx
  spec:
    ports:
    - port: 80
      name: web
    clusterIP: None
    selector:
      app: nginx
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: web-sched
  spec:
    serviceName: "nginx"
    replicas: 2
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: k8s.gcr.io/nginx-slim:0.8
          ports:
          - containerPort: 80
            name: web
          volumeMounts:
          - name: www
            mountPath: /usr/share/nginx/html
    volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        storageClassName: px-nginx-scheduled
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
  EOF


We have now created a stateful set and its associated PVCs. For each PVC, a VolumeSnapshotSchedule is created. This will trigger a snapshot based on the scheduled that we configured in the storageClass. We can view these objects by running:

.. code-block:: shell

  oc get volumesnapshotschedule


These volumeSnapshotSchedules are created when the PVC is created. We can view these events by running:

.. code-block:: shell

  oc get events --sort-by='.metadata.creationTimestamp' | grep -i snapshot