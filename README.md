# A demo configuration for Couchbase Autonomous Operator, including bi-directional XDCR, autoscaling and encryption
## Introduction and prerequisites
This document describes how to setup two Couchbase Clusters with autoscaling and encrypted XDCR, using Couchbase Autonomous Operator.

Verified and tested on a local Linux instance. As usual, Windows users are on their own.

We're going to configure two Couchbase Clusters, `cb-src` will be autoscaling and replicating the data to `cb-tgt`.

On both clusters, the admin's username is `Administrator` with a default password.

To simplify the demo, everything is going to run in the `default` name space.

Here's what you need:
* The latest version of [CAO](https://www.couchbase.com/downloads/?family=open-source-kubernetes)
* [Kubernetes tools](https://kubernetes.io/docs/tasks/tools/)
* A Kubernetes cluster of your choice

## Installing and configuring Kubernetes Cluster
### Azure AKS
Uncomment the Azure related part in both `src_cluster.yaml` and `tgt_cluster.yaml`.

Create a `<ResourceGroupName>` of your choice.

Select a `<K8sClusterName`.

`<NodeCount>>` should be >=3.

`<InstanceType>` represents the Azure VM instance type. I suggest `Standard_E8ds_v6`, as we're going to accomodate two clusters.
```
az aks create \
--resource-group <ResourceGroupName> \
--name <K8sClusterName> \
--node-count <NodeCount> \
--node-vm-size <InstanceType> \
--enable-addons monitoring \
--generate-ssh-keys
```

Update the K8s config: 
```
az aks get-credentials --resource-group <ResourceGroupName> --name <K8sClusterName>
```

You can verify that all K8s nodes are up and running with 
```
kubectl get nodes
```

### Azure lazy storage config

```
kubectl create -f az_storage.yaml
```

### AWS EKS
Choose a name for your cluster as `<K8sClusterName>`, select a `region` and run:
```
eksctl create cluster \
--name <K8sClusterName> \
--region <region> \
--fargate
```
Update the K8s config:
```
aws eks update-kubeconfig --name <K8sClusterName> --region <region>
```

You can verify that all K8s nodes are up and running with 
```
kubectl get nodes
```
### Local Kubernetes Cluster
You need a Docker runtime. On a Mac I'd suggest the following:
```
brew install colima
```
Once installed, we'll need to change the config and allow at least 10 cores and 12 GB of RAM.
```
colima start --cpu 10 --memory 12 --network-address --kubernetes
```
Then, install [k3d](https://k3d.io/) and create a cluster:
```
brew install k3d
k3d cluster create <clusterName>
```

OR

Install [Rancher Desktop](https://rancherdesktop.io/) (but needs a lot of CPU).

## Encryption setup
### Installing and configuring EasyRSA
As `cb-src` is going to use an encrypted connection to `cb-tgt` and back, we'll need to setup TLS first.

Download EasyRSA:
```
git clone https://github.com/OpenVPN/easy-rsa
cd easy-rsa/easyrsa3
```

Initialize and create the CA certificate/key. 

You will be prompted for a private key password and the CA common name (CN), something like Couchbase CA is sufficient.
```
./easyrsa init-pki
./easyrsa build-ca
```

This will produce two files, a CA cert `pki/ca.crt` and the key `pki/private/ca.key`.

We'll need to decrypt the key in order to create K8s secrets:
```
openssl rsa -in pki/private/ca.key -out pki/private/un_ca.key
```

Let's create K8s secrets containing the server and the client CA:
```
kubectl create secret tls couchbase-server-ca --cert pki/ca.crt --key pki/private/un_ca.key
```
```
kubectl create secret generic couchbase-server-xdcr --from-file=ca=pki/ca.crt
```

### Create server certificate for the source cluster
Let's create our target's server certificate. Rememeber, the cluster name is going to be `cb-src` in the `default` name space.
```
./easyrsa --subject-alt-name='DNS:*.cb-src,DNS:*.cb-src.default,DNS:*.cb-src.default.svc,DNS:*.cb-src.default.svc.cluster.local,DNS:cb-src-srv,DNS:cb-src-srv.default,DNS:cb-src-srv.default.svc,DNS:*.cb-src-srv.default.svc.cluster.local,DNS:localhost' build-server-full couchbase-server-src nopass
```

Now, let's create the K8s secret for that:
```
kubectl create secret tls src-server-tls --cert pki/issued/couchbase-server-src.crt --key pki/private/couchbase-server-src.key
```

### Create server certificate for the target cluster
Let's create our target's server certificate. Rememeber, the cluster name is going to be `cb-tgt` in the `default` name space.
```
./easyrsa --subject-alt-name='DNS:*.cb-tgt,DNS:*.cb-tgt.default,DNS:*.cb-tgt.default.svc,DNS:*.cb-tgt.default.svc.cluster.local,DNS:cb-tgt-srv,DNS:cb-tgt-srv.default,DNS:cb-tgt-srv.default.svc,DNS:*.cb-tgt-srv.default.svc.cluster.local,DNS:localhost' build-server-full couchbase-server-tgt nopass
```

Now, let's create the K8s secret for that:
```
kubectl create secret tls tgt-server-tls --cert pki/issued/couchbase-server-tgt.crt --key pki/private/couchbase-server-tgt.key
```

### Create client certificate
Now we need to create a client certificate for our `Administrator` user:
```
./easyrsa build-client-full Administrator nopass
```

According TLS secret:
```
kubectl create secret generic client-tls --from-file pki/issued/Administrator.crt --from-file pki/private/Administrator.key

```

## Install and configure CAO
### Install CAO
The installation is very simple, download the package, unpack it and change to its directory.

Copy the yaml files from this repository to the cao direcory.

### Install Custom Resource Definitions
```
kubectl apply -f crd.yaml
```

### Create admission controller
```
bin/cao create admission
```

Verify it's up and running, the pod should be in the `Running` state.
```
kubectl get pods | grep admission
``` 

### Create operator
```
bin/cao create operator
```

Verify it's up and running, the pod should be in the `Running` state.
```
kubectl get pods | grep operator | grep -v admission | awk '{print $1}'
```

## Creating Couchbase Clusters
### Creating admin secrets
Open `admin_secrets.yaml` file and change the passwords. Save the file and apply:
```
kubectl apply -f admin_secrets.yaml
```

### Installation
Please open and review the `clusters.yaml` file.

The part with the `volumeClaimTemplates` is describing AKS related setup, please feel free to change accordingly.

Change the password (encode a password with base64) and optionally change the server size settings.

Once done, install the cluster:
```
kubectl apply -f clusters.yaml
```
## Moniroring
### Logs
It's a good idea to monitor the operator's log in a dedicated window or tab.

Get the opeator's pod name and copy it.
```
kubectl get pods | grep operator | grep -v admission | awk '{print $1}'
```

Open a new terminal window and start the logs monitoring:
```
kubectl logs <operator-pod-name> -f
```
### Pods
In yet another window or tab run
```
watch "kubectl get pods"
```
You should see cluster pods being deployed, wait until all of them are ready `1/1` and `Running`.

## Buckets
### Installation
Please open and review `buckets.yaml`.

Install the buckets:
```
kubectl apply -f buckets.yaml
```
Monitor the logs for any potential issues and the pods being deployed.

### XDCR Setup
#### XDCR target cluster reference
First, for the XDCR target cluster reference, we need to identify that cluster's UUID:

```
kubectl get couchbasecluster cb-tgt -o yaml | grep clusterId
```

Copy the value.

Now open the file `xdcr_ref_src.yaml` and change the `uuid` value under `spec - xdcr` accordingly.

Once done, we need to patch our cluster config:
```
kubectl patch couchbasecluster cb-src --type merge --patch-file xdcr_ref_src.yaml
```

#### XDCR source cluster reference
First, for the XDCR source cluster reference, we need to identify that cluster's UUID:

```
kubectl get couchbasecluster cb-src -o yaml | grep clusterId
```

Copy the value.

Now open the file `xdcr_ref_tgt.yaml` and change the `uuid` value under `spec - xdcr` accordingly.

Once done, we need to patch our cluster config:
```
kubectl patch couchbasecluster cb-tgt --type merge --patch-file xdcr_ref_tgt.yaml
```

#### XDCR replication
Now it's time to spin up the replications.
Review `xdcr_replications.yaml` and apply the replication config:
```
kubectl apply -f xdcr_replications.yaml
```

### K8s port forwarding
We do everyting on the console, but it'd be nice to actually see what's going on, right?

Forward the UI port locally:
```
kubectl port-forward cb-src-0000 8091:8091
```

Open your web browser, point it to `http://localhost:8091` and authenticate with `Administrator` and the chosen password.

### Configure autoscale
The HPA is configured to scale out once the memory utilization is over 30%, we want the cluster to go above.
```
kubectl apply -f autoscaler.yaml
```

It'll need some time to stabilize, check the status with
```
kubectl describe hpa data-hpa
```

### Start pillowfight!
The `cbc-pillowfight` configuration is very agressive as we actually want to really stress the cluster.

Please open `pillowfight.yaml` and change the password.

Start the stress job:
```
kubectl apply -f pillowfight.yaml
```

Now go to the logs window and see how the operator is scaling up the cluster.

In the UI go to the source cluster's XDCR section and monitor the ongoing replication.

Once the new pod has joined the cluster and the rebalance has finished, it's safe to stop the workload - we should know by now that the autoscaling works.

Stop the stress job with
```
kubectl delete job pillowfight
```

### Optional scale in
If you'd like to verify that the scale in works too, open `autoscaler.yaml`, change `averageUtilization` to `80`, save the file and `kubectl apply -f autoscaler.yaml`.

After short period of time the HPA will scale in.

## Optional cleanup
Please remove all the not needed instances and clusters on the cloud!

### Azure AKS
```
az aks delete --name <K8sClusterName> --resource-group <ResourceGroupName>
```

### AWS EKS
Find your Fargate stack:
```
aws eks list-fargate-profiles --cluster-name <K8sClusterName> --region <region>
```
and delete it:
```
aws eks delete-fargate-profile --cluster-name <K8sClusterName> --fargate-profile-name <FARGATE_PROFILE_NAME> --region <region>
```

Find your Cloudformation stack:
```
aws cloudformation list-stacks --region <region>
```
and delete it:
```
aws cloudformation delete-stack --stack-name <STACK_NAME> --region <region>
```

Finally, delete your EKS cluster:
```
eksctl delete cluster --name <K8sClusterName> --region <region>
```
