
## Blockbridge Integration with Kubernetes

**Summary**

Blockbridge provides a Container Storage Interface ([CSI](https://github.com/container-storage-interface/spec)) driver to deliver persistent, secure, multi-tenant, cluster-accessible storage for [Kubernetes](https://kubernetes-csi.github.io/docs/). Deploy the Blockbridge CSI driver in your Kubernetes cluster using standard kubectl commands.

| Blockbridge Driver | CSI Version | stability | kubernetes version | Deploy URL |
| :---               | :---        | :---      | :---               | :---       |
| 0.2.0              | 0.2.0       | beta      | v1.10, v1.11       | https://get.blockbridge.com/kubernetes/deploy/csi/v0.2.0/csi-blockbridge.yaml |
| 1.0.0              | 1.0.0       | GA        | v1.14+             | https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-blockbridge.yaml |

As of Kubernetes version 1.14, Blockbridge supports CSI specification version 1.0.0, with Blockbridge driver version 1.0.0, as Generally Available (GA).

NOTE: If using an older version of the Blockbridge driver, please see the appropriate matching README file for version specific instructions.

| Blockbridge Driver | README version|
| :---               | :---   |
| 0.2.0              | [README.v0.2.0.md](README.v0.2.0.md) |
| 1.0.0              | [README.md (this version)](README.md) |

## Blockbridge Driver Capabilities

* Dynamic volume provisioning
* Automatic volume failover and mobility across the cluster
* Integration with RBAC for multi-namespace, multi-tenant deployments
* Quality of Service
* Encryption in-flight for control (always) and data (optional)
* Encryption at rest
* Multiple Storage Classes provide programmable, deterministic storage characteristics

## Supported Kubernetes environments

* Docker Enterprise Edition 3 (Docker EE 3.0)
* Rancher 2.X (Rancher kubernetes engine)

## Requirements

The following minimum requirements must be met to use the Blockbridge driver in Kubernetes:

* Kubernetes version 1.14 minimum
* `--allow-privileged` flag must be set to true for both the API server and the
  kubelet
* MountPropagation must be enabled (default to true since version 1.10)
* (if you use Docker) the Docker daemon of the cluster nodes must allow shared
  mounts
  
See [CSI Deploying](https://kubernetes-csi.github.io/docs/deploying.html) for more information.

## Blockbridge Driver Configuration

The driver is configured with two pieces of information:

| configuration | description |
| :----         | :----       |
| BLOCKBRIDGE_API_URL | Blockbridge controlplane API endpoint URL specified as `https://hostname.example/api` |
| BLOCKBRIDGE_API_KEY | Blockbridge controlplane access token

The API endpoint is specified as a URL, and points to the Blockbridge
controlplane. The access token authenticates the driver with the Blockbridge
controlplane.

### Create a Blockbridge account to provision volumes for Kubernetes

*NOTE: This guide assumes that the Blockbridge controlplane is running and a system password has been set. For help setting up the Blockbridge controlplane, please contact [Blockbridge Support](mailto:support@blockbridge.com).*

Create an account to hold the volumes for Kubernetes.

Use the Blockbridge CLI to create the account.

```
$ export BLOCKBRIDGE_API_HOST=blockbridge.mycompany.example
$ docker run --rm -it -e BLOCKBRIDGE_API_HOST docker.io/blockbridge/cli:latest-alpine bb --no-ssl-verify-peer account create --name kubernetes --password
```

When prompted, first enter the password for the new **kubernetes** account:
```
Enter password: 
```

When prompted, then authenticate to the Blockbridge controlplane as the system user:
```
Authenticating to https://blockbridge.mycompany.example/api

Enter user or access token: system
Password for system: 
Authenticated; token expires in 3599 seconds.
```

The account is created:
```
== Created account: kubernetes (ACT0762194C40656F03)

== Account: kubernetes (ACT0762194C40656F03)
name                  kubernetes
label                 kubernetes        
serial                ACT0762194C40656F03      
created               2018-11-19 16:15:15 +0000
disabled              no                       
```

### Create an authorization (access token) for the Blockbridge volume driver

Create an access token in the new **kubernetes** account for use as authentication for the volume driver.
```
$ export BLOCKBRIDGE_API_HOST=blockbridge.mycompany.example
$ docker run --rm -it -e BLOCKBRIDGE_API_HOST docker.io/blockbridge/cli:latest-alpine bb --no-ssl-verify-peer authorization create
```

Authenticate using the new **kubernetes** account username and password.
```
Authenticating to https://blockbridge.mycompany.example/api

Enter user or access token: kubernetes
Password for kubernetes: 
Authenticated; token expires in 3599 seconds.
```

The authorization (access token) is created.
```
== Created authorization: ATH4762194C4062668E

== Authorization: ATH4762194C4062668E
serial                ATH4762194C4062668E                    
account               kubernetes (ACT0762194C40656F03)
user                  kubernetes (USR1B62194C40656FBD)
enabled               yes                                    
created at            2018-11-19 11:15:47 -0500              
access type           online                                 
token suffix          ot50v2vA                               
restrict              auth                                   
enforce 2-factor      false                                  

== Access Token
access token          1/Nr7qLedL/P0KXxbrB8+jpfrFPBrNi3X+8H9BBwyOYg/mvOot50v2vA

*** Remember to record your access token!
```

Make a note of the access token. Use this as the BLOCKBRIDGE_API_KEY during the Blockbridge volume driver installation.

```
$ export BLOCKBRIDGE_API_KEY="1/Nr7qLedL/P0KXxbrB8+jpfrFPBrNi3X+8H9BBwyOYg/mvOot50v2vA"
```

## Driver installation in Kubernetes

### Authenticate to Kubernetes

**Authenticate with your Kubernetes cluster.**

Ensure you are authenticated to your Kubernetes cluster. A version will be printed for both the client and server successfully.

```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.2", GitCommit:"66049e3b21efe110454d67df4fa62b08ea79a19b", GitTreeState:"clean", BuildDate:"2019-05-16T16:23:09Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14+", GitVersion:"v1.14.3-docker-2", GitCommit:"7cfcb52617bf94c36953159ee9a2bf14c7fcc7ba", GitTreeState:"clean", BuildDate:"2019-06-06T16:18:13Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

NOTE: setting up `kubectl` authentication is beyond the scope of this guide. Please refer to your specific instructions for the Kubernetes service or installation you are using.

### Create a secret

Create a secret containing the Blockbridge API endpoint URL and access token.

Use BLOCKBRIDGE_API_HOST and BLOCKBRIDGE_API_KEY with the correct values for the Blockbridge controlplane, using the access token created in the **kubernetes** account.

*NOTE: While control traffic is always encrypted, we specify here to disable peer certificate verification using the `ssl-verify-peer` flag. This setting implicitly trusts the default controlplane self-signed certificate. Configuring certificate verification, including specifying custom-supplied CA certificates, is beyond the scope of this guide. Please contact [Blockbridge Support](mailto:support@blockbridge.com) for more information.*

```
$ cat > secret.yml <<- EOF
apiVersion: v1
kind: Secret
metadata:
  name: blockbridge
  namespace: kube-system
stringData:
  api-url: "https://${BLOCKBRIDGE_API_HOST}/api"
  access-token: "$BLOCKBRIDGE_API_KEY"
  ssl-verify-peer: "false"
EOF
```

Example:

```
$ cat secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: blockbridge
  namespace: kube-system
stringData:
  api-url: "https://blockbridge.mycompany.example/api"
  access-token: "1/Nr7qLedL/P0KXxbrB8+jpfrFPBrNi3X+8H9BBwyOYg/mvOot50v2vA"
  ssl-verify-peer: "false"
```

Create the secret in Kubernetes:

```
$ kubectl create -f ./secret.yml
secret "blockbridge" created
```

Ensure the secret exists in the `kube-system` namespace:

```
$ kubectl -n kube-system get secrets blockbridge
NAME          TYPE      DATA      AGE
blockbridge   Opaque    3         2m
```

### Deploy the Blockbridge Driver

Deploy the Blockbridge Driver as a DaemonSet / StatefulSet using `kubectl`.

NOTE: Choose a specific version as needed for your environment.

| Blockbridge Driver | CSI Version | stability | kubernetes version | Deploy URL |
| :---               | :---        | :---      | :---               | :---       |
| 0.2.0              | 0.2.0       | beta      | v1.10, v1.11       | https://get.blockbridge.com/kubernetes/deploy/csi/v0.2.0/csi-blockbridge.yaml |
| 1.0.0              | 1.0.0       | GA        | v1.14+             | https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-blockbridge.yaml |

Apply the appropriate version of the driver:

```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-blockbridge.yaml
```

Confirm everything was created:

```
csidriver.storage.k8s.io/eps.csi.blockbridge.com created
storageclass.storage.k8s.io/blockbridge-gp created
storageclass.storage.k8s.io/blockbridge-tls created
statefulset.apps/csi-blockbridge-controller created
serviceaccount/csi-blockbridge-controller-sa created
clusterrole.rbac.authorization.k8s.io/csi-blockbridge-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-blockbridge-provisioner-binding created
clusterrole.rbac.authorization.k8s.io/csi-blockbridge-attacher-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-blockbridge-attacher-binding created
daemonset.apps/csi-blockbridge-node created
serviceaccount/csi-blockbridge-node-sa created
clusterrole.rbac.authorization.k8s.io/csi-blockbridge-node-driver-registrar-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-blockbridge-node-driver-registrar-binding created
```

The Blockbridge CSI Driver is deployed using the [recommended mechanism](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#recommended-mechanism-for-deploying-csi-drivers-on-kubernetes) of deploying CSI drivers on Kubernetes.

### Ensure Driver is Operational

```
$ kubectl -n kube-system get pods -l role=csi-blockbridge
```

Blockbridge CSI pods are all Running:
```
$ kubectl -n kube-system get pods -l role=csi-blockbridge
NAME                                    READY     STATUS    RESTARTS   AGE
csi-blockbridge-controller-0            3/3       Running   0          6s
csi-blockbridge-node-4679b              2/2       Running   0          5s
```

### Storage Classes

A default "general purpose" StorageClass called `blockbridge-gp` is created. This is the **default** StorageClass for dynamic provisioning of storage volumes. The general purpose StorageClass provisions using the default Blockbridge storage template configured in the Blockbridge controlplane. 

There are a variety of additional storage class configuration options available, including:

1. Using transport encryption (tls)
2. Using a custom tag-based query
3. Using a named service template
4. Using explicitly specified provisioned IOPS

Several example storage classes are shown in `csi-storageclass.yaml`. Download, edit, and apply additional storage classes as needed.

```
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-storageclass.yaml
$ cat csi-storageclass.yaml
###########################################################
# StorageClass General Purpose (default)
#
# set default storage class with no additional parameters
###########################################################
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: blockbridge-gp
  namespace: kube-system
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: com.blockbridge.csi.eps
###########################################################
```

```
$ kubectl apply -f ./csi-storageclass.yaml
```
```
storageclass.storage.k8s.io "blockbridge-gp" configured
```

### Test and verify: PersistentVolumeClaim

Blockbridge storage volumes are now available via Kubernetes persistent volume claims (PVC).

Create a PersistentVolumeClaim. This dynamically provisions a volume in Blockbridge and makes it accessible to applications.

Create the PVC:
```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/examples/csi-pvc.yaml
```
```
persistentvolumeclaim "csi-pvc-blockbridge" created
```

Alternatively, download the example volume yaml, modify as needed, and apply:
```
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/examples/csi-pvc.yaml
```
```
$ cat csi-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc-blockbridge-example
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: blockbridge-gp
```
```
$ kubectl apply -f ./csi-pvc.yaml
```
```
persistentvolumeclaim "csi-pvc-blockbridge" created
```

Ensure the PVC was created successfully:
```
$ kubectl get pvc csi-pvc-blockbridge
```
```
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
csi-pvc-blockbridge   Bound    pvc-6cb93ab2-ec49-11e8-8b89-46facf8570bb   5Gi        RWO            blockbridge-gp   4s
```

### Test and verify: Application

Create a Pod (application) that utilizes the example volume. When the Pod is created, the volume will be attached, formatted and mounted, making it available to the specified application.

Create the application:
```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/examples/csi-app.yaml
```
```
pod "blockbridge-demo" created
```

Alternatively, download the application yaml, modify as needed, and apply:
```
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/examples/csi-app.yaml
```
```
$ cat csi-app.yaml
---
kind: Pod
apiVersion: v1
metadata:
  name: blockbridge-demo
spec:
  containers:
    - name: my-frontend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-bb-volume
      command: [ "sleep", "1000000" ]
    - name: my-backend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-bb-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-bb-volume
      persistentVolumeClaim:
        claimName: csi-pvc-blockbridge
```

```
$ kubectl apply -f ./csi-app.yaml
```
```
pod "blockbridge-demo" created
```

Check if the pod is running successfully:

```
$ kubectl get pod blockbridge-demo
```
```
NAME               READY     STATUS    RESTARTS   AGE
blockbridge-demo   2/2       Running   0          13s
```

### Test and Verify: Write Data

Write inside the app container:

```
$ kubectl exec -ti blockbridge-demo -c my-frontend /bin/sh
```
```
/ # df /data
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/blockbridge/2f93beb2-61eb-456b-809e-22e27e4f73cf
                       5232608     33184   5199424   1% /data

/ # touch /data/hello-world
/ # exit
```
```
$ kubectl exec -ti blockbridge-demo -c my-backend /bin/sh
```
```
/ # ls /data
hello-world
```

## Troubleshooting

### App stuck in ContainerCreating due to failed Mount

#### Symptom
Check the app status
```
$ kubectl get pod/blockbridge-demo
NAME               READY   STATUS              RESTARTS   AGE
blockbridge-demo   0/2     ContainerCreating   0          20s

$ kubectl describe pod/blockbridge-dmo
Events:
  Type     Reason                  Age   From                            Message
  ----     ------                  ----  ----                            -------
  Normal   Scheduled               10s   default-scheduler               Successfully assigned default/blockbridge-demo to kubelet.localnet
  Normal   SuccessfulAttachVolume  10s   attachdetach-controller         AttachVolume.Attach succeeded for volume "pvc-71c37e84-b302-11e9-a93f-0242ac110003"
  Warning  FailedMount             1s    kubelet, kubelet.localnet       MountVolume.MountDevice failed for volume "pvc-71c37e84-b302-11e9-a93f-0242ac110003" : rpc error: code = Unknown desc = runtime_error: /etc/iscsi/initiatorname.iscsi not found; ensure 'iscsi-initiator-utils' is installed.
```

#### Resolution

* Ensure the host running the kubelet has iSCSI client support installed on the host/node
* For CentOS/RHEL, install: `iscsi-initiator-utils` package on the host/node running the kubelet.
    * `yum install iscsi-initiator-utils`
* For Ubuntu, install: `open-iscsi-utils` package on the host/node running the kubelet.
    * `apt install open-iscsi-utils`
* Delete/re-create the application pod to retry

### PVC unauthorized

#### Symptom
Check the PVC describe output:
```
$ kubectl describe pvc csi-pvc-blockbridge
```

Provisioning failed due to "unauthorized" because the authorization access token is not valid. Ensure the correct access token is entered in the secret.

```
  Warning  ProvisioningFailed    6s (x2 over 19s)  com.blockbridge.csi.eps csi-provisioner-blockbridge-0 2caddb79-ec46-11e8-845d-465903922841  Failed to provision volume with StorageClass "blockbridge-gp": rpc error: code = Internal desc = unauthorized_error: unauthorized: unauthorized
```
 
#### Resolution

* Edit `secret.yml` and ensure correct access token and API URL are set.
* delete secret: `kubectl delete -f secret.yml`
* create secret: `kubectl create -f secret.yml`
* remove old configuration:
   * `kubectl delete -f https://get.blockbridge.com/kubernetes/deploy/examples/csi-pvc.yaml`
   * `kubectl delete -f https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-blockbridge.yaml`
* re-apply configuration:
   * `kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-blockbridge.yaml`
   * `kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/examples/csi-pvc.yaml`
   
### PVC StorageClass not found
#### Symptom
Check the PVC describe output:
```
$ kubectl describe pvc csi-pvc-blockbridge
```

Provisioning failed because the storage class specified was invalid.
```
  Warning  ProvisioningFailed  7s (x3 over 10s)  persistentvolume-controller  storageclass.storage.k8s.io "blockbridge-gp" not found
```

#### Resolution
Ensure the StorageClass exists with the same name.
```
$ kubectl get storageclass blockbridge-gp
Error from server (NotFound): storageclasses.storage.k8s.io "blockbridge-gp" not found
```

* Create the storageclass: 
```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-storageclass.yaml
```

* Alternatively, download and edit the desired storageclass:
```
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/csi/v1.0.0/csi-storageclass.yaml
$ edit csi-storageclass.yaml
$ kubectl -f apply ./csi-storageclass.yaml
```

The PVC continues to retry and picks up the storage class change:
```
$ kubectl get pvc
NAME                    STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
csi-pvc-blockbridge     Bound     pvc-6cb93ab2-ec49-11e8-8b89-46facf8570bb   5Gi        RWO            blockbridge-gp     4s
```

### APP stuck Pending cannot find Persistent Volume Claim (PVC)
#### Symptom
The app is stuck in pending:
```
$ kubectl get pod blockbridge-demo
NAME               READY     STATUS    RESTARTS   AGE
blockbridge-demo   0/2       Pending   0          14s
```

The PVC cannot be found:
```
$ kubectl describe pod blockbridge-demo

Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  12s (x6 over 28s)  default-scheduler  persistentvolumeclaim "csi-pvc-blockbridge" not found
```

#### Resolution
Ensure the PVC is created and valid.
```
$ kubectl get pvc csi-pvc-blockbridge
Error from server (NotFound): persistentvolumeclaims "csi-pvc-blockbridge" not found
```

Create the PVC:
```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/examples/csi-pvc.yaml
persistentvolumeclaim "csi-pvc-blockbridge" created
```

App retries automatically and succeeds in starting:
```
$ kubectl describe pod blockbridge-demo
  Normal   Scheduled               8s                 default-scheduler                  Successfully assigned blockbridge-demo to aks-nodepool1-56242131-0
  Normal   SuccessfulAttachVolume  8s                 attachdetach-controller            AttachVolume.Attach succeeded for volume "pvc-5332e169-ec4f-11e8-8b89-46facf8570bb"
  Normal   SuccessfulMountVolume   8s                 kubelet, aks-nodepool1-56242131-0  MountVolume.SetUp succeeded for volume "default-token-bx8b9"
```

## Where to go from here

Contact [Blockbridge Support](mailto:support@blockbridge.com) for any additional information or troubleshooting help.
