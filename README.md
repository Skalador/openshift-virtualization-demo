# OpenShift Virtualization Demo

This repository is currently under construction.

## Prerequisites

ODF due to:
- RWX block storage for live migration
- Object storage for backups. In this example we use Noobaa. 

# TL;DR

## Setup VMs

```sh
# OpenShift Virtualization
oc create -f operators/virtualization/operator-virtualization.yaml
# HyperConvergedController CR
oc apply -f operators/virtualization/hyperconverged.yaml

# Create VMs
oc new-project demo-vm
oc apply -f vms

# Wait until the VMs have successfully passed the readiness checks:
oc get virtualmachineinstance,service,route
NAME                                           AGE   PHASE     IP            NODENAME                 READY
virtualmachineinstance.kubevirt.io/demo-vm-1   32m   Running   10.133.2.66   worker-cluster-zqf4j-2   True
virtualmachineinstance.kubevirt.io/demo-vm-2   32m   Running   10.132.2.28   worker-cluster-zqf4j-1   True

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/my-service   ClusterIP   172.31.64.107   <none>        8080/TCP   29m

NAME                                HOST/PORT                                                        PATH   SERVICES     PORT   TERMINATION   WILDCARD
route.route.openshift.io/my-route   my-route-demo-vm.apps.cluster-zqf4j.dynamic.redhatworkshops.io          my-service   8080                 None
```

## Interact with VMs

```sh
virtctl -n demo-vm console demo-vm-1

```

```sh
oc -n demo-vm get vmi
NAME                   AGE     PHASE     IP            NODENAME                 READY
demo-vm-1              4h23m   Running   10.133.2.81   worker-cluster-zqf4j-2   True
demo-vm-2              4h23m   Running   10.132.2.28   worker-cluster-zqf4j-1   True

oc -n demo-vm get endpoints
NAME         ENDPOINTS                           AGE
my-service   10.132.2.28:8080,10.133.2.81:8080   4h20m

endpoint=$(oc get route -n demo-vm my-route  -ojsonpath='{.spec.host}')
oc -n openshift-monitoring exec -it alertmanager-main-0 -- curl http://${endpoint} 
This is demo VM 2 :)
```

Migrate `demo-vm-1` from worker `worker-cluster-zqf4j-2` to worker `worker-cluster-zqf4j-3`
```sh
oc pods
NAME                                       READY   STATUS    RESTARTS   AGE
virt-launcher-demo-vm-1-wpb7n              1/1     Running   0          74m
virt-launcher-demo-vm-2-rlkkq              1/1     Running   0          4h24m

oc get vmi      
NAME                   AGE     PHASE     IP            NODENAME                 READY
demo-vm-1              4h25m   Running   10.133.2.81   worker-cluster-zqf4j-2   True
demo-vm-2              4h25m   Running   10.132.2.28   worker-cluster-zqf4j-1   True

virtctl migrate demo-vm-1
VM demo-vm-1 was scheduled to migrate

oc get vm
NAME                   AGE     STATUS      READY
demo-vm-1              4h27m   Migrating   True
demo-vm-2              4h27m   Running     True

oc get pods
NAME                                       READY   STATUS    RESTARTS   AGE
virt-launcher-demo-vm-1-scjmg              0/1     Running   0          17s
virt-launcher-demo-vm-1-wpb7n              1/1     Running   0          76m
virt-launcher-demo-vm-2-rlkkq              1/1     Running   0          4h27m

oc get virtualmachineinstancemigrations
NAME                        PHASE       VMI
kubevirt-migrate-vm-w2pw6   Running     demo-vm-1

oc get virtualmachineinstancemigrations
NAME                        PHASE       VMI
kubevirt-migrate-vm-w2pw6   Succeeded   demo-vm-1

oc get po
NAME                                       READY   STATUS      RESTARTS   AGE
virt-launcher-demo-vm-1-scjmg              0/1     Running     0          3m34s
virt-launcher-demo-vm-1-wpb7n              0/1     Completed   0          80m
virt-launcher-demo-vm-2-rlkkq              1/1     Running     0          4h30m

oc get vmi
NAME                   AGE     PHASE     IP            NODENAME                 READY
demo-vm-1              4h30m   Running   10.135.0.27   worker-cluster-zqf4j-3   True
demo-vm-2              4h30m   Running   10.132.2.28   worker-cluster-zqf4j-1   True
```

## Setup Backups

https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/installing/installing-oadp-ocs.html

```sh
# OADP
oc create -f operators/oadp/operator-oadp.yaml

# Create OBC
oc create -f operators/oadp/obc-oadp.yaml

# Provision DataProtectionApplication
NAMESPACE=openshift-adp
OBCNAME=oadp-backups

export ACCESS_KEY=$(oc -n $NAMESPACE get secret $OBCNAME -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
export SECRET_KEY=$(oc -n $NAMESPACE get secret $OBCNAME -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
export BUCKET_NAME=$(oc -n $NAMESPACE get cm $OBCNAME -ojson | jq -r '.data.BUCKET_NAME')

cat << EOF > ./credentials-velero
[default]
aws_access_key_id=${ACCESS_KEY}
aws_secret_access_key=${SECRET_KEY}
EOF

oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=credentials-velero
sed -i "s/bucket:.*/bucket: ${BUCKET_NAME}/g" operators/oadp/dataprotectionapplication.yaml
oc apply -f operators/oadp/dataprotectionapplication.yaml
```

## Create Backup

https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/oadp-creating-backup-cr.html

```sh
oc apply -f snapshot-restore/backup-demo-vm-1.yaml
oc -n openshift-adp exec deployment/velero -c velero -- ./velero backup logs backup-demo-vm-1
```

## Delete VM

```sh
oc -n demo-vm delete vm demo-vm-1
```

## Restore VM

https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/restoring-applications.html

```sh
oc apply -f snapshot-restore/restore-demo-vm-1.yaml
```

## Create scheduled backups

TBD

https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/oadp-scheduling-backups-doc.html
