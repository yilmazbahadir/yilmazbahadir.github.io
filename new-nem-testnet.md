# New NEM Testnet on Kubernetes

Ready to get your NEM testnet network running on Kubernetes? This guide will walk you through the process using Helm Charts.

First of all let me briefly explain what Kubernetes and Helm are.

Kubernetes(abbreviated as K8s [but why?](https://kubernetes.io/docs/concepts/overview/))) is a powerful open-source system for automating the deployment, scaling, and management of containerized applications. It provides a platform for deploying, running, and managing containers on clusters of servers, allowing developers to easily build, deploy, and manage complex, scalable applications without worrying about the underlying infrastructure. With Kubernetes, you can build, deploy, and manage applications in a consistent, reliable, and scalable way, making it an essential tool for modern cloud-native development.

[Helm]([https://helm.sh/](https://helm.sh/)) is a package manager for Kubernetes that allows developers to easily package, configure, and deploy applications and services on Kubernetes clusters. Helm Charts are pre-configured packages of Kubernetes resources that can be deployed as a group. This makes it easy to reuse and share common configurations and simplifies the process of deploying complex applications on Kubernetes. Using Helm Charts can help save time and effort when deploying applications on Kubernetes, and they also make it easy to maintain and update applications over time.

# Setting up the environment
## Requirements
- [Docker v20.10.21](https://www.docker.com/) as a Containerization environment
- [Kubernetes v1.21](https://kubernetes.io/docs/setup/) as a container orchestration platform
- [Helm v3](https://helm.sh/docs/intro/install/) as a Kubernetes package manager
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) to run commands against Kubernetes clusters.



## How to install?

### Docker & Kubernetes

For the development and testing purposes, options are:

1. (recommended) installing [Docker Desktop](https://www.docker.com/products/docker-desktop/) which has a built-in Kubernetes support (you just need to enable K8s in the settings and it will also install kubectl for you)
2. install Docker, [minikube](https://minikube.sigs.k8s.io/docs/start/) and [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) separately.

For production purposes, the options are:

1. using a managed Kubernetes service, such as Google Kubernetes Engine (GKE), Amazon Elastic Kubernetes Service (EKS), or Azure Kubernetes Service (AKS). These services allow you to easily create and manage a cluster of nodes without having to worry about the underlying infrastructure.
2. use a tool like [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) to set up a cluster on your own servers or virtual machines. This option gives you more control over the installation process and allows you to customise your cluster to your specific needs.

Since this article is focusing on deployment of the `nis-client` to local environment so please note that you’ll need more research for production environment setup.

### Helm

Please check out [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/) to see the options to install.

### Kubernetes Controllers

- [K8s Ingress Nginx Controller](https://github.com/kubernetes/ingress-nginx)

    In order to be able to receive requests from outside of the cluster one of the options is to enable ingress in the Helm Chart values(we’ll delve into it in the following sections) and this will require an Ingress Controller to be present in the current cluster. For local setup we need to install `Ingress NGINX Controller`. (for more options please check out the [Ingress Controllers list](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/))

    1. create a file named `values-nginx-controller.yaml` in the current directory with the following content:
        ```yaml
        # configure the tcp configmap
        tcp:
          7778: localhost:7778

        # enable the service and expose the tcp ports.
        # be careful as this will potentially make them
        # available on the public web
        controller:
          service:
            enabled: true
            ports:
              http: 7890
              https: 7891
            targetPorts:
              http: http
              https: https
        ```
    2. run the following commands:
        ```bash
        helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
        helm repo update
        helm install ingress-nginx ingress-nginx/ingress-nginx --create-namespace --namespace=ingress-nginx -f ./values-nginx-controller.yaml
        ```
- [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)

  The local path provisioner in Kubernetes is a storage provisioner that allows users to create and manage persistent storage volumes backed by local storage on the nodes in a cluster.

  To install in your kubernetes cluster, run the following command in the current directory:

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.23/deploy/local-path-storage.yaml
  ```

### (Optional) Build `nis-client` docker image from source
By default helm package will automatically pull the required image from [nemofficial/nis-client](https://hub.docker.com/repository/docker/nemofficial/nis-client) Docker Hub repo.


But if you like to build from [source code](https://github.com/NemProject/nem/blob/dev/infra/docker/Dockerfile), you need to run the following commands.
```bash
git clone https://github.com/NemProject/nem
cd nem
git checkout dev
cd ./infra/docker
docker build -t nis-client .
```

# Deployment

## Pull `nem-helm-charts` package
Currently `nem-helm-charts` helm package is not published to a public registery so we need to check out [nem-helm-charts](https://github.com/yilmazbahadir/nem-helm-charts) repository into our local.


```
mkdir -p workdir-new-nem-testnet
cd workdir-new-nem-testnet
git clone https://github.com/yilmazbahadir/nem-helm-charts.git
cd nem-helm-charts
````

## Generate Nemesis block and node configurations

We need to use [nemesis-generator](https://github.com/NemProject/nemesis-generator) to generate a custom nemesis file for the initial block and the network configuration.

```bash
cd ../workdir-new-nem-testnet
git clone https://github.com/NemProject/nemesis-generator.git
cd nemesis-generator
```
You will need Pyhton3 to be installed in your local(check the version with `python3 --version`). To install follow these [instructions](https://docs.python-guide.org/starting/install3/linux/) to install on Linux (or search online for other OSs).

### Install dependencies
```bash
python3 -m pip install -r requirements.txt
```
### Generate the configuration for the generator
Please note that **in order to nodes can harvest properly, total supply should be around 9B**. That's why we're setting balance to 450M for each one of the 20 accounts below.

```bash
python3 -m configuration_generator --count 20 --seed 450000000000000 --network-name testnet --output nemesis.yaml --accounts-output user.yaml
```
### Generate NEM nemesis binary

```bash
python3 -m generator --input ./nemesis.yaml --output nemesis.bin
```

### Generate NEM node configuration
Create `nodes.yaml` file for the node information in the following format:
```yaml
nodes:
  - host: localhost-node1
    name: node1
  - host: localhost-node2
    name: node2
```
Since this is a local deployment you don't have to own a publicly available domain name.\
To set the `localhost-node1` and `localhost-node2` aliases in your local, follow these steps:
1. Get your local network IP by running `ifconfig`(linux, mac) or `ipconfig`(windows)
   ```bash
   ifconfig
   # find the IP in the output, should be something like 192.168.0.12
   ```
2. set the aliases
   ```
   sudo vi /etc/hosts
   # add the following lines, save and quit
   # Please replace 192.168.0.12 with your actual internal network IP below
   192.168.0.12 localhost-node1
   192.168.0.12 localhost-node2
   ```
Note that **using** `127.0.0.1` instead of your network IP **won't work!** So please use your internal network IP.


And run the following command to generate node configuration to `./output` folder:

```bash
python3 -m node_configuration_generator --accounts-file user.yaml --nodes-file nodes.yaml --nemesis-file ./nemesis.yaml --seed ./nemesis.bin --network-friendly-name ship --output-path ./output
```
Output files should be generated in the following structure:
```
./output
  |--node1
     |--config.user.properties
     |--nemesis.bin
     |--peers-config_testnet_ship.json
  |--node2
     |--config.user.properties
     |--nemesis.bin
     |--peers-config_testnet_ship.json
```


## Helm configurations

In order to apply custom configuration we need to create the values yaml files that will override some of the default values and pass it to the helm install command.

We'll use the files created in the previous step(nemesis generation & node config).

Copy `nemesis-generator/output` folder into the `nem-helm-charts` directory.
```bash
cd workdir-new-nem-testnet/nem-helm-charts
cp -R ../nemesis-generator/output .
```
### Prepare base64 encoded nemesis file
Encode nemesis binary file with Base64 to a (nemesis-base64.)txt file (it will be same for all nodes)
```bash
base64 -i output/node1/nemesis.bin -o nemesis-base64.txt
```
We will pass this text file as a value(--set-file) to `helm install` command in the following sections.

### Prepare the values.yaml file for nodes

For `node1` create a file named `values-node1.yaml` (inside workdir-new-nem-testnet/nem-helm-charts) with the following content, pay attention to the comments and update the file:
```yaml
config:
  user:
    nis.bootKey: # nis.bootKey from ./output/node1/config-user.properties
    nis.bootName: # nis.bootName from ./output/node1/config-user.properties
    nem.host: # nem.host from ./output/node1/config-user.properties
    nis.ipDetectionMode: Disabled

    nem.network: # nem.network from ./output/node1/config-user.properties
    nem.network.version: # nem.network.version from ./output/node1/config-user.properties
    nem.network.addressStartChar: # from ./output/node1/config-user.properties
    nem.network.generationHash: # from ./output/node1/config-user.properties
    nem.network.nemesisSignerAddress: # from ./output/node1/config-user.properties
    nem.network.totalAmount: "" # amount should be btw quotes, from ./output/node1/config-user.properties
    nem.network.nemesisFilePath: custom-nemesis.bin # don't change this

ingress:
  enabled: true
  className: "nginx"
  annotations:
  hosts:
  # update the hostname below!
    - host: # nem.host from ./output/node1/config-user.properties
      paths:
        - path: /
          pathType: ImplementationSpecific
          backend:
            service:
              port: 7890
        - path: /
          pathType: ImplementationSpecific
          backend:
            service:
              port: 7891
        - path: /
          pathType: ImplementationSpecific
          backend:
            service:
              port: 7778
  tls: []
```
Replicate the same steps to create `values-node2.yaml` file(using ./output/node2/config-user.properties this time).

If further configuration is desired you can check out all the available [values](https://github.com/yilmazbahadir/nem-helm-charts/blob/main/charts/nem-client/README.md#values) in the repo.

## Install helm package and deploy nodes

### Node1:
```bash
helm install testnet-node1 ./charts/nem-client --create-namespace --namespace=testnet-node1  --set-file config.customNemesisFileBase64=./nemesis-base64.txt --set-file config.peersConfigJson=./output/node1/peers-config_testnet_ship.json -f ./charts/nem-client/values.yaml -f ./values-node1.yaml
```

### Node2:
```bash
helm install testnet-node2 ./charts/nem-client --create-namespace --namespace=testnet-node2  --set-file config.customNemesisFileBase64=./nemesis-base64.txt --set-file config.peersConfigJson=./output/node2/peers-config_testnet_ship.json -f ./charts/nem-client/values.yaml -f ./values-node2.yaml
```

After these commands, run the following commands to check the deployment is complete.

```bash
kubectl get all --namespace testnet-node1
```

```bash
kubectl get all --namespace testnet-node2
```

![kubectl-list-all.png] (./assets/images/kubectl-list-all.png)

And to verify that the nodes are running with correct configuration, run the following commands
```bash
curl http://localhost-node1:7890/node/info
curl http://localhost-node2:7890/node/info
```

To check the nodes are harvesting, you should be observing that the height is increasing around every minute by running:
```bash
curl http://localhost-node1:7890/chain/height
curl http://localhost-node2:7890/chain/height
```

## Uninstall
To uninstall the helm packages from your K8s cluster, you can run the following commands:
```bash
helm uninstall testnet-node1 --namespace=testnet-node1
```

```bash
helm uninstall testnet-node2 --namespace=testnet-node2
```

You can verify that the uninstall is successful by running:
```bash
kubectl get all --namespace testnet-node1
# No resources found in testnet-node1 namespace.
```