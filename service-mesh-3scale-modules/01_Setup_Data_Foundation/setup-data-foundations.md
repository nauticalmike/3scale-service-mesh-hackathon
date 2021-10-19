# Provision OpenShift Data Foundation

## Prerequisites

1. Have an OpenShift (OCP) v4.x running cluster
2. Have a GitHub Account

Install the OpenShift Container Storage Operator into the recommended namespace `openshift-storage`.

Label nodes where OpenShift Container Storage should be hosted. If all worker nodes should be used, the following command can be used to label them.

```
for node in $(oc get node -l node-role.kubernetes.io/worker -o name); do oc label ${node} cluster.ocs.openshift.io/openshift-storage=""; done
```

Apply the StorageCluster:

```
oc apply -f openshift-storage/StorageCluster.yaml -n openshift-storage
```

3scale requires a RWX storage class. The storage class used by the 3scale control plane will be `ocs-storagecluster-cephfs`. Ensure that the storage class is available with the following command.

```
oc get storageclasses
```
