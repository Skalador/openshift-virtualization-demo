- [OpenShift Virtualization Demo](#openshift-virtualization-demo)
  - [Prerequisites](#prerequisites)
  - [Setup Operators](#setup-operators)
    - [OpenShit Virtualization](#openshit-virtualization)
    - [OpenShift GitOps](#openshift-gitops)
    - [OADP Operator](#oadp-operator)
- [Manually interacting with OpenShift Virtualization VMs](#manually-interacting-with-openshift-virtualization-vms)
  - [Create VMs](#create-vms)
    - [Create VMs with the CLI](#create-vms-with-the-cli)
    - [Create VMs with the GUI](#create-vms-with-the-gui)
    - [Create VMs with the API](#create-vms-with-the-api)
    - [Create a Windows VM](#create-a-windows-vm)
  - [Access VMs with the `virtctl` command](#access-vms-with-the-virtctl-command)
  - [Migrate VMs](#migrate-vms)
    - [Migrate VMs with the CLI](#migrate-vms-with-the-cli)
    - [Migrate VMs with the GUI](#migrate-vms-with-the-gui)
  - [Delete VMs](#delete-vms)
    - [Delete VMs via CLI](#delete-vms-via-cli)
    - [Delete VMs via GUI](#delete-vms-via-gui)
  - [Snapshot and Restore](#snapshot-and-restore)
    - [Snapshot](#snapshot)
    - [Restore](#restore)
  - [Leverage OpenShift Features](#leverage-openshift-features)
    - [Services](#services)
    - [Routes](#routes)
    - [Health Checks](#health-checks)
    - [Hotplug of resources to a VM](#hotplug-of-resources-to-a-vm)
      - [CPU](#cpu)
      - [Memory](#memory)
- [Using OpenShift GitOps with VMs](#using-openshift-gitops-with-vms)
  - [Provision all VMs](#provision-all-vms)
  - [Start VM via Git](#start-vm-via-git)
  - [Stop VM via Git](#stop-vm-via-git)
  - [Connect from the VM to the database inside a container:](#connect-from-the-vm-to-the-database-inside-a-container)
- [OADP Backups](#oadp-backups)
  - [Create Backup](#create-backup)
  - [Delete VM](#delete-vm)
  - [Restore VM](#restore-vm)
  - [Create scheduled backups](#create-scheduled-backups)

# OpenShift Virtualization Demo

This repository consolidates multiple other repositories, which are specialized for specific tasks around OpenShift Virtualization. Consequently, this repository is continuously changing.
Some of the included repositories are:
- [GitOps and OpenShift Virtualization](https://github.com/Skalador/openshift-virt-gitops)
- [TAM Your Tech Hour - OpenShift Virtualization](https://github.com/Skalador/tam-your-tech-hour-openshift-virtualization)
- [TAM Your Tech Hour - OpenShift Virtualization deep dive](https://github.com/Skalador/tam-your-tech-hour-openshift-virtualization-part-2)


## Prerequisites

This demo requires 
- OpenShift client `oc` 
- [KubeVirt client](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/virtualization/getting-started#virt-installing-virtctl-client_virt-using-the-cli-tools) `virtctl` are required.
- [`jq`](https://jqlang.github.io/jq/)

[OpenShift Data Foundation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/storage/configuring-persistent-storage#red-hat-openshift-data-foundation) due to:
- RWX block storage for live migration
- Object storage for OADP backups. In this example we use `Noobaa`. 

## Setup Operators

### OpenShit Virtualization

```sh
# OpenShift Virtualization
oc create -f operators/virtualization/operator-virtualization.yaml
# HyperConvergedController CR
oc apply -f operators/virtualization/hyperconverged.yaml
```

### OpenShift GitOps

```sh
# GitOps Operator
oc apply -f operators/gitops/operator-gitops.yaml

# Obtain GitOps password
oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

### OADP Operator


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


# Manually interacting with OpenShift Virtualization VMs

## Create VMs

This activity will be performed in the `demo-vm` project, which does not exist yet.
```sh
oc new-project demo-vm
```

### Create VMs with the CLI

Create a VM from the provided demo VMs, i.e with a preconfigured YAML file.
```sh
oc create -f vms/virtualmachine-1.yaml 
virtualmachine.kubevirt.io/demo-vm-1 created
```

Alternatively the `virtctl` tool can be used to create a VM, e.g:
```sh
virtctl create vm \
--name=fedora-plum-sailfish-86 \
--instancetype=u1.medium \
--preference=fedora \
--volume-datasource=src:openshift-virtualization-os-images/fedora \
--cloud-init-user-data I2Nsb3VkLWNvbmZpZwpjaHBhc3N3ZDoKICBleHBpcmU6IGZhbHNlCnBhc3N3b3JkOiA0b2htLTlnZDctZzE4OAp1c2VyOiBmZWRvcmEK
```

[Link to demonstration as GIF](./src/video/create-vm-cli.gif)

### Create VMs with the GUI

With instance types: 
[Link to demonstration as GIF](./src/video/create-vm-gui-catalog-instance-types.gif)

With templates catalog:
[Link to demonstration as GIF](./src/video/create-vm-gui-catalog-template-catalog.gif)

### Create VMs with the API

Obtain the token:
```sh
TOKEN=$(oc whoami -t)
```

Obtain the `API` endpoint with either of those commands and then build the full API endpoint `APIFULL`:
```sh
API=$(oc cluster-info | grep 'running at' | awk '{print $NF}')
API=$(oc cluster-info | grep 'running at' | sed 's/.*running at //')

APIPATH="/apis/kubevirt.io/v1/namespaces/demo-vm/virtualmachines/"
APIFULL="${API}${APIPATH}"

echo $APIFULL
https://api.somedomain.io:6443/apis/kubevirt.io/v1/namespaces/demo-vm/virtualmachines/
```

Create a VM by sending the `json` data to the `APIFULL`:
```sh
curl -k -X POST -d @api/vm.json -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" https://api.somedomain.io:6443/apis/kubevirt.io/v1/namespaces/demo-vm/virtualmachines/
```

### Create a Windows VM

```sh
oc new-project win-vm

oc apply -f win-vm/
configmap/sysprep-windows-zgb4pd created
virtualmachine.kubevirt.io/windows created
```

## Access VMs with the `virtctl` command

```sh
virtctl -n demo-vm console demo-vm-1
```

## Migrate VMs

This activity will be performed in the `demo-vm` project.
```sh
oc project demo-vm
```

### Migrate VMs with the CLI

Migrate the `rhel-9-demo-vm` from `ocp4-worker2.aio.example.com` to `ocp4-worker3.aio.example.com` via the CLI:
```sh
oc get vmi
NAME              AGE     PHASE     IP            NODENAME                       READY
centos7-demo-vm   18m     Running   10.129.2.15   ocp4-worker1.aio.example.com   True
demo-vm-1         30m     Running   10.128.2.33   ocp4-worker2.aio.example.com   True
rhel-9-demo-vm    2m16s   Running   10.128.2.42   ocp4-worker2.aio.example.com   True

virtctl migrate rhel-9-demo-vm 
VM rhel-9-demo-vm was scheduled to migrate

oc get vmim
NAME                        PHASE       VMI
kubevirt-migrate-vm-kdqdz   Succeeded   rhel-9-demo-vm

oc get vmi
NAME              AGE     PHASE     IP            NODENAME                       READY
centos7-demo-vm   19m     Running   10.129.2.15   ocp4-worker1.aio.example.com   True
demo-vm-1         31m     Running   10.128.2.33   ocp4-worker2.aio.example.com   True
rhel-9-demo-vm    2m58s   Running   10.131.0.15   ocp4-worker3.aio.example.com   True
```

[Link to demonstration as GIF](./src/video/migrate-vm-cli.gif)

### Migrate VMs with the GUI

[Link to demonstration as GIF](./src/video/migrate-vm-gui.gif)

## Delete VMs

This activity will be performed in the `demo-vm` project.
```sh
oc project demo-vm
```

### Delete VMs via CLI

```sh
oc get vm
NAME              AGE   STATUS    READY
centos7-demo-vm   25m   Running   True
rhel-9-demo-vm    28m   Running   True


oc delete vm centos7-demo-vm
virtualmachine.kubevirt.io "centos7-demo-vm" deleted
```

[Link to demonstration as GIF](./src/video/delete-vm-cli.gif)

### Delete VMs via GUI

[Link to demonstration as GIF](./src/video/delete-vm-gui.gif)


## Snapshot and Restore

This activity will be performed in the `demo-vm` project.
```sh
oc project demo-vm
```

### Snapshot

[Link to demonstration as GIF](./src/video/snapshot-gui.gif)

### Restore

[Link to demonstration as GIF](./src/video/restore-gui.gif)


## Leverage OpenShift Features 

OpenShift Virtualization can use standard OpenShift features, such as
- `Services` 
- `Routes` 
- `LivenessProbes`
- `ReadinessProbes`

To showcase this behavior, create a new project called `vm-feature-showcase` and deploy all objects from the `vms/` folder.

```sh
oc new-project vm-feature-showcase

oc create -f vms/
route.route.openshift.io/my-route created
service/my-service created
virtualmachine.kubevirt.io/demo-vm-1 created
virtualmachine.kubevirt.io/demo-vm-2 created
```

This will deploy the  following components:
- Virtual machine 1 --> First web server VM
- Virtual machine 2 --> Second web server VM
- `service` --> Enables the connection to all VMs
- `route` --> Enables HTTP(S) traffic to the VMs via the `service`

```sh
oc get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/virt-launcher-demo-vm-1-5wkgj   1/1     Running   0          6m
pod/virt-launcher-demo-vm-2-rmbzq   1/1     Running   0          5m59s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/my-service   ClusterIP   172.30.228.202   <none>        8080/TCP   6m1s

NAME                                HOST/PORT                                                            PATH   SERVICES     PORT   TERMINATION   WILDCARD
route.route.openshift.io/my-route   my-route-vm-feature-showcase.apps.975cz.dynamic.redhatworkshops.io          my-service   8080                 None

NAME                                             PHASE       PROGRESS   RESTARTS   AGE
datavolume.cdi.kubevirt.io/demo-vm-1-ds-fedora   Succeeded   100.0%                6m1s
datavolume.cdi.kubevirt.io/demo-vm-2-ds-fedora   Succeeded   100.0%                6m1s

NAME                                           AGE   PHASE     IP            NODENAME                       READY
virtualmachineinstance.kubevirt.io/demo-vm-1   6m    Running   10.128.2.17   ocp4-worker2.aio.example.com   True
virtualmachineinstance.kubevirt.io/demo-vm-2   6m    Running   10.129.2.8    ocp4-worker1.aio.example.com   True

NAME                                   AGE    STATUS    READY
virtualmachine.kubevirt.io/demo-vm-1   6m1s   Running   True
virtualmachine.kubevirt.io/demo-vm-2   6m1s   Running   True
```

### Services

The IPs of the `virt-launcher-` are `10.128.2.17` and `10.129.2.8`
```sh
oc get pods -owide
NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE                           NOMINATED NODE   READINESS GATES
virt-launcher-demo-vm-1-5wkgj   1/1     Running   0          11m   10.128.2.17   ocp4-worker2.aio.example.com   <none>           1/1
virt-launcher-demo-vm-2-rmbzq   1/1     Running   0          11m   10.129.2.8    ocp4-worker1.aio.example.com   <none>           1/1
```

Both `virt-launcher-` pods are attached to the same service called `my-service` with port `8080`
```sh
oc get endpoints
NAME         ENDPOINTS                          AGE
my-service   10.128.2.17:8080,10.129.2.8:8080   11m
```

The port was specified in the `service`
```sh
oc get svc my-service -ojsonpath='{.spec.ports}' | jq
[
  {
    "port": 8080,
    "protocol": "TCP",
    "targetPort": 8080
  }
]
```

Sending `curl` requests to the `service` will then end up at any of the two attached VMs. As `service`s are not exposed via a public URL, those `service`s cannot be reach from outside the cluster. Consequently, the next subsection will take care of this.

### Routes

`Routes` are meant as an HTTP(S) endpoint for OpenShift. The `route` called `my-route` is connected to the `service` called `my-service`. The route can be reached at the `my-route-vm-feature-showcase.apps.975cz.dynamic.redhatworkshops.io` URL.
```sh
oc get route
NAME       HOST/PORT                                                            PATH   SERVICES     PORT   TERMINATION   WILDCARD
my-route   my-route-vm-feature-showcase.apps.975cz.dynamic.redhatworkshops.io          my-service   8080                 None
```

Extract the URL into the `endpoint` variable and `curl` it 10 times, to see alternating responses:

```sh
endpoint=$(oc get route my-route  -ojsonpath='{.spec.host}')
echo $endpoint
my-route-vm-feature-showcase.apps.975cz.dynamic.redhatworkshops.io

for i in {1..10}; do curl http://$endpoint; done
This is demo VM 2 :)
This is demo VM 1 :)
This is demo VM 2 :)
This is demo VM 2 :)
This is demo VM 1 :)
This is demo VM 1 :)
This is demo VM 1 :)
This is demo VM 1 :)
This is demo VM 1 :)
This is demo VM 2 :)
```

[Link to demonstration as GIF](./src/video/route-service.gif)


### Health Checks

Both VMs have `readinessProbes` in place, which will wait `180` seconds after startup before checking the availability of the webserver on port `8080`
```sh
oc get vm demo-vm-1 -ojsonpath='{.spec.template.spec.readinessProbe}' | jq
{
  "failureThreshold": 10,
  "httpGet": {
    "port": 8080
  },
  "initialDelaySeconds": 180,
  "periodSeconds": 20,
  "successThreshold": 3,
  "timeoutSeconds": 10
}
```

### Hotplug of resources to a VM

The hotplug availability is described in [solution article 7080814](https://access.redhat.com/solutions/7080814).

#### CPU

CPU hotplug requires OpenShift 4.16 or later. The Hotplug process is described in the [upstream KubeVirt project](https://kubevirt.io/user-guide/compute/cpu_hotplug/#hotplug-process)

Verify the currently available CPUs from the OpenShift perspective:

```sh
oc get vm demo-vm-1 -ojson | jq '.spec.template.spec.domain.cpu'
{
  "cores": 1,
  "sockets": 1,
  "threads": 1
}
```

Verify the currently available CPUs from the Guest OS perspective:
```sh
virtctl console demo-vm-1

lscpu | head -n 5
Architecture:                         x86_64
CPU op-mode(s):                       32-bit, 64-bit
Address sizes:                        48 bits physical, 48 bits virtual
Byte Order:                           Little Endian
CPU(s):                               1
```

Increase the number of CPUs via CLI:

```sh
oc patch vm demo-vm-1 --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/domain/cpu/sockets", "value": 2}]'
```

Wait for the live migration to complete:

```sh
oc get vmim --watch
NAME                             PHASE     VMI
kubevirt-workload-update-49zkb   Scheduling   demo-vm-1
kubevirt-workload-update-49zkb   Running   demo-vm-1
kubevirt-workload-update-49zkb   Succeeded   demo-vm-1
```

Verify the currently available CPUs from the OpenShift perspective:

```sh
oc get vm demo-vm-1 -ojson | jq '.spec.template.spec.domain.cpu.sockets'                                                  
2
```

Verify the currently available CPUs from the Guest OS perspective:
```sh
virtctl console demo-vm-1

lscpu | head -n 5
Architecture:                         x86_64
CPU op-mode(s):                       32-bit, 64-bit
Address sizes:                        48 bits physical, 48 bits virtual
Byte Order:                           Little Endian
CPU(s):                               2
```

#### Memory

The Hotplug process is described in the [upstream KubeVirt project](https://kubevirt.io/user-guide/compute/memory_hotplug/)
RFE for Memory Hotplug: [CNV-32072](https://issues.redhat.com/browse/CNV-32072)

# Using OpenShift GitOps with VMs

This part of the demo will create four `namespaces`
- `dev-demo-db`  -> Database for development
- `prod-demo-db` -> Database for production
- `dev-demo-vm`  -> VMs for development
- `prod-demo-vm` -> VMs for production

These `namespaces` (and other resources) are deployed via [OpenShift GitOps](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.14/html/understanding_openshift_gitops/index), which is based on the open source project [Argo CD](https://argoproj.github.io/cd/). The `namespaces`, i.e. multi-stage environments, are managed via [`kustomize`](https://kustomize.io/).
The overlay structure is as follows:
```sh
tree applicationsets 
.
├── demo-db
│   ├── applicationset-demo-db.yaml
│   └── kustomize
│       ├── base
│       │   ├── deployment.yaml
│       │   ├── kustomization.yaml
│       │   └── service.yaml
│       └── overlays
│           ├── dev
│           │   └── kustomization.yaml
│           └── prod
│               └── kustomization.yaml
└── demo-vm
    ├── applicationset-demo-vm.yaml
    └── kustomize
        ├── base
        │   ├── kustomization.yaml
        │   ├── route.yaml
        │   ├── service.yaml
        │   ├── virtualmachine-1.yaml
        │   └── virtualmachine-2.yaml
        └── overlays
            ├── dev
            │   └── kustomization.yaml
            └── prod
                └── kustomization.yaml
```

The database `namespaces` are pretty similar, as they just have a `Deployment` and `Service`, each.

The VM `namespaces` hold the following base configuration:
- Virtual machine 1 -> First web server VM
- Virtual machine 2 -> Second web server VM
- `service`         -> Enables the connection to all VMs 
- `route`           -> Enables HTTP(S) traffic to the VMs via the `service`
The `namespaces` differ in the overlays, because the development environment `dev-demo-vm` has the second VM stopped, as there is no need for it.

## Provision all VMs

```sh
# Obtain GitOps password
oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d

# Create ApplicationSets
oc apply -f applicationsets/demo-vm/applicationset-demo-vm.yaml
oc apply -f applicationsets/demo-db/applicationset-demo-db.yaml

# Apply permissions for namespaces
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n dev-demo-vm
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n prod-demo-vm
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n dev-demo-db
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n prod-demo-db
```

## Start VM via Git

[Link to demonstration as GIF](./src/video/start_vm_via_git.gif)

## Stop VM via Git

[Link to demonstration as GIF](./src/video/stop_vm_via_git.gif)

## Connect from the VM to the database inside a container:
```sh
oc -n dev-demo-db get svc
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
dev-my-db   ClusterIP   X.X.X.X         <none>        3306/TCP   133m

virtctl -n dev-demo-vm console dev-demo-vm-1

mysqlshow -u developer -pdeveloper -h dev-my-db.dev-demo-db.svc.cluster.local
+--------------------+
|     Databases      |
+--------------------+
| information_schema |
| performance_schema |
| sampledb           |
+--------------------+

exit
```

[Link to demonstration as GIF](./src/video/connect_vm_to_database_container.gif)

# OADP Backups

Work in progress. TBD.

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
