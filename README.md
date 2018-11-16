
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
* Quality of Service guarantees
* Built-in transport encryption (TLS) capabilities and always-on data encryption at rest
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
| BLOCKBRIDGE_API_URL | Blockbridge controlplane API endpoint URL |
| BLOCKBRIDGE_ACCESS_TOKEN | Blockbridge controlplane access token

The API endpoint is specified as a URL, and points to the Blockbridge
controlplane. The access token authenticates the driver with the Blockbridge
controlplane.

For help determining your API endpoint or generating an access token, please contact [Blockbridge Support](mailto:support@blockbridge.com).

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

Replace BLOCKBRIDGE_API_URL and BLOCKBRIDGE_ACCESS_TOKEN with the correct values for the Blockbridge controlplane.

```
$ cat > secret.yml <<- EOF
apiVersion: v1
kind: Secret
metadata:
  name: blockbridge
  namespace: kube-system
stringData:
  api-url: '{{ BLOCKBRIDGE_API_URL }}'
  access-token: '{{ BLOCKBRIDGE_ACCESS_TOKEN }}'
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
  access-token: "1/AKFm669............brr1XWWg"
```

Create the secret in Kubernetes:

```
$ kubectl create -f ./secret.yml
secret "blockbridge" created
```

Ensure the secret exists in the `kube-system` namespace:

```
$ kubectl -n kube-system get secrets
blockbridge                                      Opaque                                2         1h
```

### Deploy the Blockbridge Driver

Deploy the Blockbridge Driver as a DaemonSet / StatefulSet using `kubectl`.

The latest version of the driver is applied:

```
$ kubectl apply -f https://get.blockbridge.com/kubernetes/deploy/csi/latest/csi-blockbridge.yaml
```

NOTE: the `latest` release tracks the latest Blockbridge driver release. Choose a specific version if needed for your environment.

| Blockbridge Driver | CSI Specification Version | Deploy URL |
| :---               | :---                      | :---       |
| latest             | 0.2.0                     | https://get.blockbridge.com/kubernetes/deploy/csi/latest/csi-blockbridge.yaml |
| 0.1.4              | 0.2.0                     | https://get.blockbridge.com/kubernetes/deploy/csi/0.1.4/csi-blockbridge.yaml |

The Blockbridge CSI Driver is deployed using the [recommended mechanism](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#recommended-mechanism-for-deploying-csi-drivers-on-kubernetes) of deploying CSI drivers on Kubernetes.

### Storage Classes

A default "general purpose" StorageClass called `blockbridge-gp` is created. This is the **default** StorageClass for dynamic provisioning of storage volumes. The general purpose StorageClass provisions using the default Blockbridge storage template configured in the Blockbridge controlplane. 

There are a variety of additional storage class configuration options available, including:


Additional configuration options include:

1. Using transport encryption (tls)
2. Using a custom tag-based query
3. Using a named service template
4. Using explicitly specified provisioned IOPS

Several example storage classes are shown in `csi-storageclass.yaml`. Download, edit, and apply additional storage classes as needed.

```
$ curl -OsSL https://get.blockbridge.com/kubernetes/deploy/csi/csi-storageclass.yaml
```

Edit `csi-storageclass.yaml` as desired, and apply the new storage classes:

```
$ kubectl apply -f csi-storageclass.yaml
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
        name: my-blockbridge-volume
      command: [ "sleep", "1000000" ]
    - name: my-middleend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-blockbridge-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-blockbridge-volume
      persistentVolumeClaim:
        claimName: csi-pvc-blockbridge-example

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
$ kubectl exec -ti blockbridge-demo -c my-middleend /bin/sh
/ # ls /data
hello-world
```

