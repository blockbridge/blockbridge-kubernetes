
## Blockbridge Integration with Kubernetes

**Summary**

Blockbridge provides a Container Storage Interface ([CSI](https://github.com/container-storage-interface/spec)) driver to deliver persistent, secure, multi-tenant, cluster-accessible storage for Kubernetes. Deploy the Blockbridge CSI driver in your Kubernetes cluster as a DaemonSet/StatefulSet using standard Kubernetes commands.

| csi version supported | stability | kubernetes version |
| :---              | :---      | :--- |
| [v0.2.0](https://github.com/container-storage-interface/spec/blob/v0.2.0/spec.md) | beta | v1.10+

The CSI specification is currently in **beta** in Kubernetes. As such, the Blockbridge  driver is currently in **beta**. It is expected to reach General Availability (GA) in Kubernetes 1.13.

## Blockbridge Driver Capabilities

* Dynamic volume provisioning
* Automatic volume failover and mobility across the cluster
* Integration with RBAC for multi-namespace, multi-tenant deployments
* Quality of Service
* Encryption in-flight for control (always) and data (optional)
* Encryption at rest
* Multiple Storage Classes provide programmable, deterministic storage characteristics

## Supported Kubernetes environments

* AKS (azure kubernetes service)
* EKS (amazon elastic container service for kubernetes)
* GKE (google kubernetes engine)
* Rancher 2.X (rancher kubernetes engine)
* Vanilla / Custom Kubernetes

## Requirements

The following minimum requirements must be met to use the Blockbridge driver in Kubernetes:

* Kubernetes version 1.10 minimum
* `--allow-privileged` flag must be set to true for both the API server and the
  kubelet (default in some environemnts: GCE, GKE, etc.)
* MountPropagation must be enabled (default to true since version 1.10)
* (if you use Docker) the Docker daemon of the cluster nodes must allow shared
  mounts
  
See [CSI Setup](https://kubernetes-csi.github.io/docs/Setup.html) for more information.

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

Authenticate using the new **kubernetes** account username and password.

```
$ export BLOCKBRIDGE_API_HOST=blockbridge.mycompany.example
$ docker run --rm -it -e BLOCKBRIDGE_API_HOST docker.io/blockbridge/cli:latest-alpine bb --no-ssl-verify-peer authorization create
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
account               kubernetes-srust2 (ACT0762194C40656F03)
user                  kubernetes-srust2 (USR1B62194C40656FBD)
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

## Driver installation in Kubernetes

### Authenticate to Kubernetes

**Authenticate with your Kubernetes cluster.**

Ensure you are authenticated to your Kubernetes cluster. A version will be printed for both the client and server successfully.

```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.3", GitCommit:"2bba0127d85d5a46ab4b778548be28623b32d0b0", GitTreeState:"clean", BuildDate:"2018-05-21T09:17:39Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.3", GitCommit:"2bba0127d85d5a46ab4b778548be28623b32d0b0", GitTreeState:"clean", BuildDate:"2018-05-21T09:05:37Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
```

### Create a secret

Create a secret containing the Blockbridge API endpoint URL and access token.

Replace BLOCKBRIDGE_API_URL and BLOCKBRIDGE_API_KEY with the correct values for the Blockbridge controlplane, using the access token created in the **kubernetes** account.

*NOTE: While control traffic is always encrypted, we specify here to disable peer certificate verification using the `ssl-verify-peer` flag. This setting implicitly trusts the default controlplane self-signed certificate. Configuring certificate verification, including specifying custom-supplied CA certificates, is beyond the scope of this guide. Please contact [Blockbridge Support](mailto:support@blockbridge.com) for more information.*

```
$ cat > secret.yml <<- EOF
apiVersion: v1
kind: Secret
metadata:
  name: blockbridge
  namespace: kube-system
stringData:
  api-url: "{{ BLOCKBRIDGE_API_URL }}"
  access-token: "{{ BLOCKBRIDGE_API_KEY }}"
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

Apply the latest version of the driver:

```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/csi/latest/csi-blockbridge.yaml
```

Confirm everything was created:

```
storageclass.storage.k8s.io "blockbridge-gp" created
serviceaccount "blockbridge-csi-attacher" created
clusterrole.rbac.authorization.k8s.io "blockbridge-external-attacher-runner" created
clusterrolebinding.rbac.authorization.k8s.io "blockbridge-csi-attacher-role" created
service "csi-attacher-blockbridge" created
statefulset.apps "csi-attacher-blockbridge" created
serviceaccount "blockbridge-csi-provisioner" created
clusterrole.rbac.authorization.k8s.io "blockbridge-external-provisioner-runner" created
clusterrolebinding.rbac.authorization.k8s.io "blockbridge-csi-provisioner-role" created
service "csi-provisioner-blockbridge" created
statefulset.apps "csi-provisioner-blockbridge" created
serviceaccount "csi-blockbridge" created
clusterrole.rbac.authorization.k8s.io "csi-blockbridge" created
clusterrolebinding.rbac.authorization.k8s.io "csi-blockbridge" created
daemonset.apps "csi-blockbridge" created
```

NOTE: the `latest` release tracks the latest Blockbridge driver release. Choose a specific version if needed for your environment.

| Blockbridge Driver | CSI Specification Version | Deploy URL |
| :---               | :---                      | :---       |
| latest             | 0.2.0                     | https://get.blockbridge.com/kubernetes/deploy/csi/latest/csi-blockbridge.yaml |
| 0.1.4              | 0.2.0                     | https://get.blockbridge.com/kubernetes/deploy/csi/v0.1.4/csi-blockbridge.yaml |

The Blockbridge CSI Driver is deployed using the [recommended mechanism](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#recommended-mechanism-for-deploying-csi-drivers-on-kubernetes) of deploying CSI drivers on Kubernetes.

### Ensure Driver is Operational

```
$ kubectl -n kube-system get pods
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
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/csi/latest/csi-storageclass.yaml
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

Alternatively, download the example volume yaml, modify as needed, and apply:
```
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/examples/csi-pvc.yaml
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

### Test and verify: Application

Create a Pod (application) that utilizes the example volume. When the Pod is created, the volume will be attached, formatted and mounted, making it available to the specified application.

Create the application:
```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/examples/csi-app.yaml
```

Alternatively, download the application yaml, modify as needed, and apply:
```
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/examples/csi-app.yaml
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

Check if the pod is running successfully:

```
$ kubectl describe pods/blockbridge-demo
```

### Test and Verify: Write Data

Write inside the app container:

```
$ kubectl exec -ti blockbridge-demo -c my-frontend /bin/sh
/ # touch /data/hello-world
/ # exit
$ kubectl exec -ti blockbridge-demo -c my-backend /bin/sh
/ # ls /data
hello-world
```

