# Running CockroachDB in Multi-Tenanted Kubernetes Cluster

Is this demo CockroachDB is configured to run as a DbaaS configuration. What this means is that multiple instances of CockroachDB running in the same kubernetes cluster. Nodes are allocated to specific workers within the Kubernetes cluster with the use of node labels and node selectors.

Deploy and Kubernetes cluster with nine worker nodes.

Create some variables to help with the deployment. We will be creating three separate tenants in a single cluster.

```
loc1="tenant1"
loc2="tenant2"
loc3="tenant3"
```


Tag your nodes with a label for each tenant. In the example below I have tagged my AKS node with three different tags. (tenant=1, tenant=2, tenant=3)

```ssh
kubectl label nodes aks-nodepool-61388275-vmss000000 tenant=1
kubectl label nodes aks-nodepool-61388275-vmss000001 tenant=1
kubectl label nodes aks-nodepool-61388275-vmss000002 tenant=1

kubectl label nodes aks-nodepool-61388275-vmss000003 tenant=2
kubectl label nodes aks-nodepool-61388275-vmss000004 tenant=2
kubectl label nodes aks-nodepool-61388275-vmss000005 tenant=2

kubectl label nodes aks-nodepool-61388275-vmss000006 tenant=3
kubectl label nodes aks-nodepool-61388275-vmss000007 tenant=3
kubectl label nodes aks-nodepool-61388275-vmss000008 tenant=3
```

Now that the node are tagged we are able to deploy CockroachDB three separate times into the cluster. First, we will create three separate namespaces.

Create a namespace for each of the tenants
```
kubectl create namespace $loc1
kubectl create namespace $loc2
kubectl create namespace $loc3
```

Create certificates for each of the three clusters.

Create the folder structure for the certificates.

```
mkdir certs my-safe-directory certs/tenant1 certs/tenant2 certs/tenant3 my-safe-directory/tenant1 my-safe-directory/tenant2 my-safe-directory/tenant3
```

All communication between the node and clients is secure. The simplest way to achieve this is with the built in certificate authority. We can use any machine with the cockroach binary installed to create all the required certificates. Create the CA certificate and key pair:

```
cockroach cert create-ca \
--certs-dir=certs/tenant1 \
--ca-key=my-safe-directory/tenant1/ca.key
```

Create a client certificate and key pair for the root user:

```
cockroach cert create-client \
root \
--certs-dir=certs/tenant1 \
--ca-key=my-safe-directory/tenant1/ca.key
```

Upload the client certificate and key to the Kubernetes cluster as a secret.

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs/tenant1 \
--namespace $loc1
```

Create the certificate and key pair for your CockroachDB nodes in the first tenant:

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$loc1 \
cockroachdb-public.$loc1.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$loc1" \
"*.cockroachdb.$loc1.svc.cluster.local" \
--certs-dir=certs/tenant1 \
--ca-key=my-safe-directory/tenant1/ca.key
```

Upload it as a kubernetes secret.

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs/tenant1 \
--namespace $loc1
```

Repeat the process for tenant 2. Create the CA certificate and key pair:

```
cockroach cert create-ca \
--certs-dir=certs/tenant2 \
--ca-key=my-safe-directory/tenant2/ca.key
```

Create a client certificate and key pair for the root user:

```
cockroach cert create-client \
root \
--certs-dir=certs/tenant2 \
--ca-key=my-safe-directory/tenant2/ca.key
```

Upload the client certificate and key to the Kubernetes cluster as a secret.

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs/tenant2 \
--namespace $loc2
```

Create the certificate and key pair for your CockroachDB nodes in the second tenant:

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$loc2 \
cockroachdb-public.$loc2.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$loc2" \
"*.cockroachdb.$loc2.svc.cluster.local" \
--certs-dir=certs/tenant2 \
--ca-key=my-safe-directory/tenant2/ca.key
```

Upload it as secret.

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs/tenant2 \
--namespace $loc2
```

To complete the process, complete for the final tenant. Create CA and key.

```
cockroach cert create-ca \
--certs-dir=certs/tenant3 \
--ca-key=my-safe-directory/tenant3/ca.key
```

Create client certificate and key.

```
cockroach cert create-client \
root \
--certs-dir=certs/tenant3 \
--ca-key=my-safe-directory/tenant3/ca.key
```

Upload as a kubernetes secret.

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs/tenant3 \
--namespace $loc3
```

Create the certificate and key pair for your CockroachDB nodes in the second tenant:

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$loc3 \
cockroachdb-public.$loc3.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$loc3" \
"*.cockroachdb.$loc3.svc.cluster.local" \
--certs-dir=certs/tenant3 \
--ca-key=my-safe-directory/tenant3/ca.key
```

Upload it as secret 

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs/tenant3 \
--namespace $loc3
```


All of the certificates are created so we can now deploy the CockroachDB pods into each namesapce in the cluster.

Apply the manifests.
```
kubectl -n $loc1 apply -f ./manifests/tenant1/tenant1.yaml
kubectl -n $loc2 apply -f ./manifests/tenant2/tenant2.yaml
kubectl -n $loc3 apply -f ./manifests/tenant3/tenant3.yaml
```

In each of these manifest files a `nodeSelector` has been configured to pin the pods to the nodes with the matching labels.

Example:
```
    spec:
      nodeSelector:
        tenant: '1'
```


To verify that the pods are running on the correct nodes in the cluster run the flowing commands.

```
kubectl get po -n tenant1 -o wide
kubectl get po -n tenant2 -o wide
kubectl get po -n tenant3 -o wide
```

To ensure security between tenants network policies can be created to prevent traffic between these pods.

