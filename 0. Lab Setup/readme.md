# 0. Lab Setup

## Environment

This workbook is designed around a kubernetes cluster with Calico Enterprise installed.
Current requirements:
* 3 nodes kubernetes cluster:
* master1 (10.0.1.20) 
* worker1 (10.0.1.30)
* worker2 (10.0.1.31)

kubernetes version: 1.24.9
Calico Enterprise version: 3.15


We will be using an example microservices based application call Online Boutique or Hipstershop as the basis for all labs and examples. This application will be deployed in its own namespace within the cluster.

For testing, we will also be deploying a tools container called Network-Multitool which has built in tools for testing and troubleshooting within the cluster.

## Setup kubernetes cluster

Please follow the instruction provided here to create a kubernetes cluster with the following IP address ranges:
```bash
--pod-network-cidr=10.48.0.0/16 --service-cidr=10.49.0.0/16
```
To connect the Lynx lab environment to Calico Cloud, first we need to install Project Calico as the CNI. This will enable networking in the cluster.

### Install Calico Enterprise

For more details on the install of Project Calico, refer to the [Online Documentation](https://docs.tigera.io/v3.15/getting-started/kubernetes/generic-install).


First, we will install the Tigera Operator on the cluster:

```bash
kubectl create -f https://projectcalico.docs.tigera.io/archive/v3.22/manifests/tigera-operator.yaml
```

Next, we apply the custom resource manifest to install Calico as the CNI.

```yaml
kubectl apply -f -<<EOF
# This section includes base Calico Enterprise installation configuration.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Install Calico Enterprise
  variant: TigeraSecureEnterprise
  # List of image pull secrets to use when installing images from a container registry.
  # If specified, secrets must be created in the `tigera-operator` namespace.
  imagePullSecrets:
    - name: tigera-pull-secret
  # Optionally, a custom registry to use for pulling images.
  # registry: <my-registry>
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.48.0.0/24
      encapsulation: None
      natOutgoing: Enabled
      nodeSelector: all()
---
# This section configures the Tigera web manager.
# Remove this section for a Managed cluster.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Manager
apiVersion: operator.tigera.io/v1
kind: Manager
metadata:
  name: tigera-secure
spec:
  # Authentication configuration for accessing the Tigera manager.
  # Default is to use token-based authentication.
  auth:
    type: Token
---
# This section installs and configures the Calico Enterprise API server.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: tigera-secure
---
# This section installs and configures Calico Enterprise compliance functionality.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Compliance
apiVersion: operator.tigera.io/v1
kind: Compliance
metadata:
  name: tigera-secure
---
# This section installs and configures Calico Enterprise intrusion detection functionality.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.IntrusionDetection
apiVersion: operator.tigera.io/v1
kind: IntrusionDetection
metadata:
  name: tigera-secure
---
# This section configures the Elasticsearch cluster used by Calico Enterprise.
# Remove this section for a Managed cluster.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.LogStorage
apiVersion: operator.tigera.io/v1
kind: LogStorage
metadata:
  name: tigera-secure
spec:
  nodes:
    count: 1
---
# This section configures collection of Tigera flow, DNS, and audit logs.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.LogCollector
apiVersion: operator.tigera.io/v1
kind: LogCollector
metadata:
  name: tigera-secure
---
# This section configures Prometheus for Calico Enterprise.
# For more information, see: https://docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Monitor
apiVersion: operator.tigera.io/v1
kind: Monitor
metadata:
  name: tigera-secure
---
EOF
```

After a couple of minutes, all nodes should now show as Ready.
```bash
kubectl get nodes
```
```bash
NAME      STATUS   ROLES           AGE     VERSION
master1   Ready    control-plane   4d18h   v1.24.9
worker1   Ready    <none>          3d13h   v1.24.9
worker2   Ready    <none>          3d13h   v1.24.9
```

## Deploying the Application

**Create a Namespace for the application**

First, let's create a namespace called 'hipstershop' for the application:

```bash
kubectl create namespace hipstershop
```

**Deploy the application**

Next we will deploy the Online Boutique (Hipstershop) application to the namespace. This will install the application from the Google repository.

```bash
kubectl apply -n hipstershop -f hipstershop.yaml
```


**Verify Pods are Running**

After deploying the application, wait for all pods to come online in the namespace, this will take a couple of minutes.
```bash
kubectl get pods -n hipstershop
```

The output should look similar to below.
```bash
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-776f7f9bb-z7pjc                1/1     Running   0          50s
cartservice-55cd86d896-5mxh4             1/1     Running   0          51s
checkoutservice-5944bf5f96-dqrf5         1/1     Running   0          52s
currencyservice-5478cb46cd-ss22v         1/1     Running   0          51s
emailservice-97556899b-ntr69             1/1     Running   0          52s
frontend-746879fd55-t9hnz                1/1     Running   0          52s
loadgenerator-6cdf76b6d4-wvg64           1/1     Running   0          51s
paymentservice-6b74ff9f78-pknsj          1/1     Running   0          52s
productcatalogservice-86ddd945bb-9zmh8   1/1     Running   0          51s
recommendationservice-955dccff4-mp6j6    1/1     Running   0          52s
redis-cart-5b569cd47-cm8tt               1/1     Running   0          50s
shippingservice-7784dcb6c8-p6pms         1/1     Running   0          51s
```


**Deploy Network-MultiTool Pod in the default namespace**

The Network-MultiTool pod will be used in two namespaces to test the created network policies.

First, deploy the MultiTool into the default namespace:

```bash
kubectl run multitool --image=wbitt/network-multitool
```

**Deploy a second copy of Network-Multitool to the hipstershop namespace**

Next, deploy a copy of the Mutlitool into the hipstershop namespace:

```bash
kubectl run multitool -n hipstershop --image=wbitt/network-multitool
```

Verify that both pods are up and running:

```bash
kubectl get pods -A | grep multitool
```

Both pods should be Running:

```bash
default                      multitool                                        1/1     Running            0              12s
hipstershop                  multitool                                        1/1     Running            0              31m
```

**Test Connectivity to frontend-external service or nodeport**

First, get the IP address of the frontend-external service:
```bash
kubectl get svc -n hipstershop frontend-external
```
```bash
NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
frontend-external   NodePort   10.49.112.50   <none>        80:30007/TCP   6s
```

Next, connect to the shell inside of the Network-Multitool pod in default namespace using kubectl exec:
```bash
kubectl exec multitool --stdin --tty -- /bin/bash
```

Now we'll use curl to verify we can reach the web frontend of the application (your **Cluster-IP** address might be different in your cluster, replace it accordingly):
```bash
bash-5.1# curl -I 10.49.14.192
HTTP/1.1 200 OK
Set-Cookie: shop_session-id=1939f999-1237-4cc7-abdb-949423eae483; Max-Age=172800
Date: Wed, 26 Jan 2022 20:14:20 GMT
Content-Type: text/html; charset=utf-8
```
Exit the shell by typing **exit**.

### Access the Online Boutique application

Now that the application is deployed and running, let's try to access it by reaching out the frontend service that has been exposed. you should be able to reach your Online Boutique application at:
```yaml
http://10.0.1.20:30007/
```

![hipstershop](images/hipstershop.png)

## Reference Documentation

[Github - Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo)

[Github - Network-Multitool Pod](https://github.com/wbitt/Network-MultiTool)
