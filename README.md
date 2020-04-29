# Detailed installation guide for the HPE CSI Driver for Kubernetes using the HPE Primera and 3PAR Container Storage Provider

[TOC]

Before we go any further, lets scope this guide and detail who this guide is for and what are the requirements to get started so you can follow along. This guide is geared to a Kubernetes admin who needs to configure the cluster for Persistent Storage using an HPE Primera or 3PAR array. We will cover some basics in the beginning on how to setup and configure `kubectl` to communicate with the cluster after which we will deploy the HPE CSI Driver for Kubernetes and the HPE 3PAR and Primera Container Storage Provider. We will also validate the installation with several examples and highlight some troubleshooting tips in case you run into any issues. 

This guide assumes that you are comfortable working on the command-line. Much of the work done to manage Kubernetes clusters is done at the command line like issuing `kubectl` commands and heavy file manipulation with `vim` or `nano`.

## Getting Started
This guide starts with an existing Kubernetes cluster. This can be a fresh install with `kubeadm` or kubespray. The deployment of Kubernetes is outside of the scope of this document. If you don't have a cluster up and running, I recommend that you get started there.

    - [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
    - [kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/)

### Configuring kubectl

Configure `kubectl` on your laptop or workstation to communicate with the Kubernetes cluster, as none of our work will be done directly on the cluster hosts. Please refer to [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). 

To help you get started I will go through the configuration of `kubectl` on Linux. `kubectl` is also available for macOS and Windows.

  1. Download the latest version of `kubectl`.

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl`
```

  2. Set the executable flag on the `kubectl` binary.

```
chmod +x ./kubectl
```

  3. Move `kubectl` into your PATH.

```
sudo mv ./kubectl /usr/local/bin/kubectl
```

  4. Test to make sure it is working correctly.

```
kubectl version
```

>**NOTE** <br />
>`kubectl` is also available for **macOS** with [Homebrew](https://brew.sh/) `brew install kubectl` or **Windows** using [Chocolatey](https://chocolatey.org/install): `choco install kubernetes-cli`.

<br />

The next steps of configuring `kubectl` is configuring the **.kube/config** file that will enable `kubectl` to communicate with your Kubernetes cluster. In order to communicate with the Kubernetes cluster, `kubectl` looks for a file named config in the **$HOME/.kube** directory. You can specify other kubeconfig files by setting the `KUBECONFIG` environment variable or by setting the `--kubeconfig` flag. 

To do this, start by logging into one of your master nodes and switch to root:

```
ssh k8s-admin@kube-master
sudo su -
```

We need to copy the contents from the .kube/config to our local machine. Make sure to get the entire file starting at `apiVersion` to the end.

```
[root@kube-master ~]# cat .kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    <contents>
    server: https://192.168.1.10:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: 
    <contents>
    client-key-data: 
    <contents>
```

Back on our laptop or workstation, create the `.kube` folder within your user profile.

- Linux/Mac: /home/<user>/.kube

```
[chris@workstation ~]# mkdir .kube
```

- Windows: C:\Users\<user>\.kube

```
C:\Users\chris>mkdir .kube
```

Create the **.kube/config** and paste the contents from the .kube/config file into this file

```
vi .kube/config
```

Paste
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    <contents>
    server: https://192.168.1.10:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: 
    <contents>
    client-key-data: 
    <contents>
```
Save and exit.

- Check that `kubectl` and the **.kube\config** file are configured properly by getting the cluster state:

```
kubectl cluster-info
Kubernetes master is running at https://192.168.1.10:6443
coredns is running at https://192.168.1.10:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
kubernetes-dashboard is running at https://192.168.1.10:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
If you see a URL response, `kubectl` is correctly configured to access your cluster.

> If you get an error when running `kubectl cluster-info`, double-check that you copied the contents of the **.kube/config** correctly.

Now lets look at the nodes within our cluster.

```
kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
kube-master    Ready    master   40d   v1.17.4
```

You can also list any available pods. (If this is a new cluster, you will not see any listed here.)

```
kubectl get pods

NAME                         READY   STATUS    RESTARTS   AGE
nginx-pod-5bb4787f8d-7ndj6   1/1     Running   0          6m39s
```

If this is a new cluster, you will more than likely see the following:

```
No resources found in default namespace.
```

> **Namespaces:** Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces. Namespaces are intended for use in environments with many users spread across multiple teams, or projects. Namespaces are a way to divide cluster resources between multiple users. The **kube-system** namespace is used for Kubernetes components managed by Kubernetes.

If you want to view pods from a different namespace, use the **-n <namespace>** option:

```
kubectl get pods -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5b644bc49c-m6l7s          1/1     Running   0          40d
calico-node-dvmfq                                 1/1     Running   0          40d
coredns-6955765f44-7k6ld                          1/1     Running   0          40d
coredns-6955765f44-g6hpn                          1/1     Running   0          40d
etcd-kube-master.virtware.io                      1/1     Running   0          40d
kube-apiserver-kube-master.virtware.io            1/1     Running   0          40d
kube-controller-manager-kube-master.virtware.io   1/1     Running   0          40d
kube-proxy-5k9pp                                  1/1     Running   0          40d
kube-scheduler-kube-master.virtware.io            1/1     Running   0          40d
```

Now we know **kubectl** is configured and able to talk to our Kubernetes cluster.

### Installing Helm

The next steps will walk through configuring a packagem management tool for Kubernetes. Helm helps simplify the process of deploying apps including the HPE CSI Driver and the HPE 3PAR and Primera CSP. Helm helps you manage Kubernetes applications — Helm Charts help you define, install, and upgrade even the most complex Kubernetes application.

For more information, please refer to [https://helm.sh/docs/](https://helm.sh/docs/)

> **IMPORTANT**
> The HPE 3PAR and Primera CSP only supports Helm 3. The HPE 3PAR and Primera CSP relies upon Kubernetes CRDs which has limited support within Helm 2.

#### From the Binary Releases
Every release of Helm provides binary releases for a variety of OSes. These binary versions can be manually downloaded and installed.

1. Download your desired version. https://github.com/helm/helm/releases/latest
2. Unpack it (`tar -zxvf helm-v3*.tgz`)
3. Find the `helm` binary in the unpacked directory, and move it to its desired destination (`mv linux-amd64/helm /usr/local/bin/helm`)

From there, you should be able to run the client: `helm help`.

>Note!
>If you are using macOS, you can use [Homebrew](https://brew.sh/) `brew install helm` or Windows using [Chocolatey](https://chocolatey.org/install): `choco install kubernetes-helm`.

Once Helm is installed, you can add a chart repository. One popular starting location is the official Helm stable charts:

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
"stable" has been added to your repositories
```

Once this is installed, you will be able to list the charts you can install:

```
helm search repo stable
NAME                                    CHART VERSION   APP VERSION             DESCRIPTION
stable/acs-engine-autoscaler            2.2.2           2.1.1                   DEPRECATED Scales worker nodes within agent pools
stable/aerospike                        0.3.2           v4.5.0.5                A Helm chart for Aerospike in Kubernetes
stable/airflow                          6.7.2           1.10.4                  Airflow is a platform to programmatically autho...
stable/ambassador                       5.3.1           0.86.1                  A Helm chart for Datawire Ambassador
... and many more
```

To install a chart, you can run `helm install <chart name>`. Helm is **namespace** aware so depending on the chart, you may need to use the **-n <namespace>** parameter.

```
helm install stable/nginx-ingress --generate-name
NAME: nginx-ingress-1588033585
LAST DEPLOYED: Mon Apr 27 17:26:29 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The nginx-ingress controller has been installed.
```

Run `helm ls` to see what charts have been deployed

```
helm ls
NAME                        NAMESPACE    REVISION    UPDATED                                 STATUS      CHART                  APP VERSION
nginx-ingress-1588033585    default      1           2020-04-27 17:26:29.1835012 -0700 PDT   deployed    nginx-ingress-1.36.3   0.30.0
```

To uninstall a chart, run `helm uninstall <chart name>`.

Now that we have Helm configured, lets deploy the HPE CSI driver.

## Deploying the HPE CSI Driver and HPE 3PAR and Primera Container Storage Provider with Helm

To get started with the deployment of the HPE CSI Driver, check out the HPE Storage Container Orchestrator Documentation (SCOD for short) site. SCOD is an umbrella documentation project for all Kubernetes and Docker integrations for HPE primary storage tailored for IT Ops, developers and partners. Including HPE 3PAR and Primera, HPE Cloud Volumes and HPE Nimble Storage. 

The HPE CSI Driver is deployed by using industry standard means, either a Helm chart or an Operator. For this demo, we will be using Helm to the deploy the CSI driver.

The official Helm chart for the HPE CSI Driver for Kubernetes is hosted on [hub.helm.sh](hub.helm.sh). There you will find the configuration and installation instructions for the chart.

The first step of installing the HPE CSI Driver is creating the **values.yaml** file.

Please refer to this sample [values.yaml](https://github.com/hpe-storage/co-deployments/tree/master/helm/values/csi-driver) file.

```
vi primera-values.yaml
```

Copy the following into the file. Make sure to set the **backendType: primera3par** and the **backend** to the array IP along with the array username and password.

> **NOTE:** <br />
> The user specified will need at minimum the **edit** role on the array.

```
# Default values for hpe-csi-storage.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# image pull policy for all images in csi deployment
imagePullPolicy: 'Always'

# HPE backend storage type (nimble, primera3par)
backendType: primera3par

secret:
  # parameters for specified backendType (nimble, primera3par)
  create: true
  backend: 192.168.1.10
  username: 3paradm
  password: 3pardata
  servicePort: "8080"

# flavor
flavor: kubernetes

# log level for all csi driver components
logLevel: info

## For creating the StorageClass automatically:
storageClass:
  create: false

  ## Set StorageClass as the default StorageClass
  ## Ignored if storageClass.create is false
  defaultClass: false

  ## Set a StorageClass name
  ## Ignored if storageClass.create is false
  name: hpe-standard
  # Enable volume resize
  ## Ignored if storageClass.create is false
  allowVolumeExpansion: true
  # Default storage class parameters
  parameters:
    fsType: xfs
    volumeDescription: "Volume created by the HPE CSI Driver for Kubernetes"
    accessProtocol: "iscsi"

# CRD creation controls
## For HELM 3 CRD's are handled automatically without this flag.
## This flag should be enabled only for HELM 2
crd:
  nodeInfo:
    create: false
  volumeInfo:
    create: false
```
Save and exit.

> **IMPORTANT** <br />
> Deploying the HPE CSI Driver with the HPE 3PAR and Primera CSP, currently doesn't support the creation of the **default** `StorageClass` in the Helm chart. Make sure to set **create: false** or omit the StorageClass section.

### Installing the chart
To install the chart with the name hpe-csi:

Add HPE helm repo:

```
helm repo add hpe https://hpe-storage.github.io/co-deployments
helm repo update
```

Install the latest chart:

```
helm install hpe-csi hpe/hpe-csi-driver --namespace kube-system -f primera-values.yaml
```

Wait a few minutes as the deployment finishes.

You can verify that everything is up and running correctly with the listing out the pods.

```
kubectl get pods -n kube-system | grep "hpe\|csp"
hpe-csi-controller-84d8569476-vt7xg                5/5       Running   0          13m
hpe-csi-node-s4c8z                                 2/2       Running   0          13m
primera3par-csp-66f775b555-2qclg                   1/1       Running   0          13m
```

```
kubectl get secret -n kube-system | grep primera3par
primera3par-secret                Opaque                                5         13m
```

If all of the components show in `Running` state then we have successfully deployed the HPE CSI driver and the HPE 3PAR and Primera Container Storage Provider.

> **IMPORTANT** <br /> If you see the `primera3par-csp` showing `CrashLoopBackOff`, double-check the array IP and credentials used in the Helm chart. If there is a mistake you will need to uninstall the Helm chart and redeploy with the updated **values.yaml**.

## Using the HPE CSI Driver and HPE 3PAR and Primera Container Storage Provider

Now let's validate the deployment by creating a `StorageClass`, `PersistentVolumeClaim` and a Deployment.

In this tutorial, we will create a `StorageClass` API object using the CSI driver and the HPE 3PAR Primera CSP but you have a few more parameters that must be included relevant to the CSI driver and `Secret` for the **primera3par** backend. For a full list of supported parameters, please refer to the [CSP specific documentation](https://scod.hpedev.io/container_storage_provider/hpe_3par_primera/index.html).

The below YAML declarations are meant to be created with `kubectl create`. Either copy the content to a file on the host where `kubectl` is being executed, or copy & paste into the terminal, like this:

```
kubectl create -f-
< paste the YAML >
^D (CTRL + D)
```

Create a StorageClass API object for a Primera Data Reduction volume.

```markdown
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: primera-reduce-sc
provisioner: csi.hpe.com
parameters:
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/provisioner-secret-name: primera3par-secret
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
  csi.storage.k8s.io/controller-publish-secret-name: primera3par-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: kube-system
  csi.storage.k8s.io/node-publish-secret-name: primera3par-secret
  csi.storage.k8s.io/node-publish-secret-namespace: kube-system
  csi.storage.k8s.io/node-stage-secret-name: primera3par-secret
  csi.storage.k8s.io/node-stage-secret-namespace: kube-system
  # Uncommment for k8s 1.15 for resize support
  csi.storage.k8s.io/controller-expand-secret-name: primera3par-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: kube-system
  cpg: "SSD_r6"
  provisioning_type: "reduce"
  accessProtocol: "fc"
```

Create a `PersistentVolumeClaim`. This object creates a `PersistentVolume` as defined, make sure to reference the correct `.spec.storageClassName`.

```
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-nginx
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: primera-reduce-sc
```

Now let's again verify the `PersistentVolume` was created successfully.

```
kubectl get pv
NAME                               CAPACITY ACCESS MODES RECLAIM POLICY STATUS  CLAIM              STORAGECLASS      AGE
persistentvolume/pvc-3c783d63-0... 50Gi     RWO          Delete         Bound   default/pvc-nginx  primera-reduce-sc 16m
```

The above output means that the HPE CSI Driver successfully provisioned a new volume based upon the **primera-reduce-sc** `StorageClass`. The volume is not attached to any node yet. It will only be attached to a node once a scheduled workload requests the `PersistentVolumeClaim` **pvc-nginx**. 

> **NOTE** <br/> If you find the `PVC` is stuck in a **Pending** state, double check the **CPG** and **provisioning_type** (full, tpvv, or reduce) are correct within the `StorageClass`. Also check the status of the CSP `kubectl get pods -n kube-system | grep "hpe\|csp"` or look at the CSP logs. <br />For more information refer to the [Diagnostics documentation](https://scod.hpedev.io/csi_driver/diagnostics.html).


Now, let us create a `Pod` that refers to the above volume. When the `Pod` is created, the volume will be attached, formatted and mounted.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-main
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-main
  template:
    metadata:
      labels:
        run: nginx-main
    spec:
      initContainers:
      - name: web-content
        image: busybox
        volumeMounts:
        - name: export
          mountPath: "/export"
        command: ["/bin/sh", "-c", 'echo "<h1><font color=green>This Pod was deployed with Persistent Storage using HPE CSI Driver and the HPE 3PAR and Primera CSP</font></h1>" > /export/index.html']
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: export
          mountPath: "/usr/share/nginx/html"
      volumes:
      - name: export
        persistentVolumeClaim:
          claimName: pvc-nginx
```

Now check to verify that the `Pod` has been created and is in **Running** state.

```
kubectl get pods
NAME                                                        READY     STATUS    RESTARTS   AGE
nginx-deploy-main-577fcb7cb4-z75m5                          1/1       Running   0          95s
```

Finally lets take a look at the nginx app. We can use `kubectl port-forward` to verify the application is working correctly, which is defined by specifying the external port mapping to the port on the pod. <external_port>:<pod_port>

```
kubectl port-forward nginx-deploy-main-577fcb7cb4-z75m5 5000:80
Forwarding from 127.0.0.1:5000 -> 80
Forwarding from [::1]:5000 -> 80
```

If all was successful, open a browser and you should see, "This Pod was deployed with Persistent Storage using HPE CSI Driver and the HPE 3PAR and Primera CSP".

Use **Ctrl+C** to exit the port-forwarding.

## Next steps
Stay tuned to HPE DEV for future blogs regarding the HPE CSI Driver for Kubernetes. In the meantime, if you want to learn more about Kubernetes, CSI and the integration with HPE storage products, you can find a ton of Resources out on [SCOD](https://scod.hpedev.io/)! Connect with us on [Slack](https://hpedev.slack.com/). We hang out in #kubernetes, #nimblestorage and #3par-primera.
