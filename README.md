
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

#### 2. Deploy the CSI plugin and sidecars:

Before you continue, be sure to checkout to a [tagged
release](https://github.com/blockbridge/csi-blockbridge/releases). For
example, to use the version `v0.1.4` you can execute the following command:

```
$ kubectl apply -f https://raw.githubusercontent.com/blockbridge/csi-blockbridge/master/deploy/kubernetes/releases/csi-blockbridge-v0.1.4.yaml
```

A new storage class will be created with the name `blockbridge-gp` which is
responsible for dynamic provisioning. This is set to **"default"** for dynamic
provisioning. If you're using multiple storage classes you might want to remove
the annotation from the `csi-storageclass.yaml` and re-deploy it. This is
based on the [recommended mechanism](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#recommended-mechanism-for-deploying-csi-drivers-on-kubernetes) of deploying CSI drivers on Kubernetes.

There are a variety of additional example commented-out storage classes in the
deployment file detailing additional configuration options:

1. Using transport encryption (tls)
2. Using a custom tag-based query
3. Using a named service template
4. Using explicitly specified provisioned IOPS

*Note that the deployment proposal to Kubernetes is still a work in progress and not all of the written
features are implemented. When in doubt, open an issue or ask #sig-storage in [Kubernetes Slack](http://slack.k8s.io)*

#### 3. Test and verify:

Create a PersistentVolumeClaim. This makes sure a volume is created and provisioned on your behalf:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: blockbridge-gp
```

After that create a Pod that refers to this volume. When the Pod is created, the volume will be attached, formatted and mounted to the specified Container

```
kind: Pod
apiVersion: v1
metadata:
  name: my-csi-app
spec:
  containers:
    - name: my-frontend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-do-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-do-volume
      persistentVolumeClaim:
        claimName: csi-pvc 
```

Check if the pod is running successfully:


```
$ kubectl describe pods/my-csi-app
```

Write inside the app container:

```
$ kubectl exec -ti my-csi-app /bin/sh
/ # touch /data/hello-world
/ # exit
$ kubectl exec -ti my-csi-app /bin/sh
/ # ls /data
hello-world
```
