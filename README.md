<img src="https://github.com/hyperglance/aws-rule-automations/blob/master/files/b5dfbb6c-75c8-493b-8c5d-d68b3272cf0f.png" alt="Hyperglance Logo" />

# Hyperglance - Kubernetes

This Repository contains a kubernetes deployment and a helm chart that can be used to deploy Hyperglance to your Kubernetes cluster.

:information_source: This README assumes that you already have a Kubernetes Cluster deployed.

## Pre-Requisites

Please follow the below steps to install the pre-requisites required to deploy Hyperglance to K8.

### Install Kubectl

Pleae follow the [Kubernetes](https://kubernetes.io/docs/tasks/tools/install-kubectl/) guide to get up and running with kubectl.

### Install Helm (Optional)

Use the following Guide to install [HELM](https://helm.sh/docs/intro/install/) .

### Clone / Download the repository

Clone the repository:

```bash
mkdir code && cd code
git clone https://github.com/hyperglance/kubernetes.git
```

Download the [Zip](https://github.com/hyperglance/kubernetes/archive/v0.2.1.zip) release, and extract it.

### Create a namespace (Optional)

You may wish to deploy Hyperglance to its own [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), this is notnormally required for small K8 deployments.

To create one, execute the following commands:

```bash
kubectl create ns hyperglance
```

## Usage - Kubectl

To deploy hyperglance using kubectl, from the kubectl directory of the cloned repository run:

```bash
# Default Namespace
kubectl apply -f hyperglance.yml

# Named Namespace
kubectl apply -f hyperglance.yml -n hyperglance
```

The deployment will create a LoadBalancer, to get the IP / DNS Name, run:

```bash
# Default Namespace
kubectl get svc

# Named Namespace
kubectl get svc -n hyperglance

Output:

NAME              TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
hyperglance       LoadBalancer   10.0.18.238    52.232.222.170   80:30750/TCP,443:30876/TCP   87m
```

Use the IP to access Hyperglance

## Usage - Helm

To deploy hyperglance using kubectl, from the helm directory of the cloned repository run:

```bash
# Default Namespace
helm install hyperglance hyperglance

# Named Namespace
helm install hyperglance hyperglance -n hyperglance

Output:

NAME: hyperglance
LAST DEPLOYED: Thu Nov  5 14:09:06 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace default svc -w hyperglance'
  export SERVICE_IP=$(kubectl get svc --namespace default hyperglance --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo http://$SERVICE_IP:

```

Use the IP to access Hyperglance