<img src="https://github.com/hyperglance/aws-rule-automations/blob/master/files/b5dfbb6c-75c8-493b-8c5d-d68b3272cf0f.png" alt="Hyperglance Logo" />

# Hyperglance - Kubernetes

This Repository contains a Kubernetes deployment and a helm chart that can be used to deploy Hyperglance to your <a href="https://docs.aws.amazon.com/eks/latest/userguide/fargate.html" target="_blank">AWS EKS using Fargate</a> cluster.

:warning: Deploying on EKS with Fargate is considerably different from a standard Kubernetes deployment and extensive prerequisites are required

## prerequisites

This deployment assumes you already have the following prerequisites satisfied:

:information_source: [This guide maybe useful in assisting you to satisfy these prerequisites](https://aws.amazon.com/blogs/containers/running-stateful-workloads-with-amazon-eks-on-aws-fargate-using-amazon-efs/)

1. A functioning AWS EKS using Fargate cluster - [AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/fargate-getting-started.html) or [EKSCTL guide](https://eksctl.io/usage/fargate-support/)
2. EKS ALB controller provisioned and configured - [AWS guide](https://aws.amazon.com/premiumsupport/knowledge-center/eks-alb-ingress-controller-fargate/)
3. Kubernetes EFS CSIDriver and Storage class deployed to the cluster - 
   If you don't have this loaded currently, you can apply this using the [csidriver.yaml](kubectl/csidriver.yaml) and [storageclass.yaml](kubectl/storageclass.yaml) files included in this repo - ```kubectl apply -f kubectl/csidriver.yaml -f kubectl/storageclass.yaml```
3. An EFS filesystem configured and accessible from your EKS cluster with [two access points configured](https://docs.aws.amazon.com/efs/latest/ug/create-access-point.html) to be used for persistent volumes - confirming the paths exist on EFS for the access points
4. A TLS certificate available in [Amazon Certificate Manager](https://aws.amazon.com/certificate-manager/) for use with the Application Load Balancer
5. An IAM OIDC identity provider is configured for your cluster, to allow service account authentication into AWS - [AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
5. Kubectl installed and configured for use with your EKS cluster - [AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)
6. Helm installed - [Helm guide](https://helm.sh/docs/intro/install/)

### Clone the repository or Download the zip

To Clone the repository:

```bash
git clone https://github.com/hyperglance/kubernetes.git
```

Or download the [ZIP](https://github.com/hyperglance/kubernetes/archive/refs/heads/master.zip) and extract it.

### Create a namespace (Optional)

You may wish to deploy Hyperglance to its own [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), this is not normally required for small K8 deployments.

To create one, execute the following commands:

```bash
kubectl create ns hyperglance
```
*If you wish to use a separate namespace, ensure you append ```-n hyperglance``` to all future kubectl commands and you have a Fargate profile setup to [allocate workloads for the namespace](https://docs.aws.amazon.com/eks/latest/userguide/fargate-profile.html)*

### Create an IAM role and policy for Hyperglance

Hyperglance will utilise the service account configured during setup, to poll AWS for resources.

Create a role, select the trusted entity as the OIDC provider created in the prerequisites and assign a policy with the permissions required for Hyperglance - [these can be found here](https://support.hyperglance.com/knowledge/aws-iam-policy-requirements)

We'll reduce the scope of this trust to only the service account we will create on EKS. On the newly created role in AWS IAM, edit the trust policy/trust relationship. It will look something like this by default:

```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.eu-west-3.amazonaws.com/id/183E80BB183AB94E102232070EDC421B"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<AWS REGION>.amazonaws.com/id/183E80BB183AB94E102232070EDC4969:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Change the line after string equals to the following, substituting  <namespace>, <service-account-username> and changing :aud to :sub:

```bash
"StringEquals": {
          "oidc.eks.<AWS REGION>.amazonaws.com/id/183E80BB183AB94E102232070EDC4969:sub": "system:serviceaccount:<namespace>:<service-account-username>"
}
```

Make a note of the ARN of the role. You will need to populate this value for your setup depending on your method of deployment:

## Usage - Helm

To deploy hyperglance using Helm, from the [helm directory](helm/) of the cloned repository:

Edit [values.yaml](helm/hyperglance/values.yaml) and populate the required values with the IAM role ARN, certificate ARN, EFS filesystem ID and EFS accesspoint values you created previously:

Helm - [values.yaml](helm/hyperglance/values.yaml)

IAM Role ARN:

```yaml
serviceAccount:
  name: hyperglance
  iamRole: <HERE> # IAM role K8 service account will use to auth against AWS
```

EFS storage:

```
efs:
  filesystem_id: fs-xxxxxxxx
  postgresql:
    accesspoint_id: fsap-xxx
  hyperglance:
    accesspoint_id: fsap-xxx

```

Load balancer certificate:

```bash
albCertificateArn: arn:aws:acm:xxxx
```

Finally, go ahead and run the following to deploy to your EKS cluster:

```bash
helm install hyperglance hyperglance -n hyperglance
```
*Allow up to 5 minutes for the pod to provision and become available*

## Usage - Kubectl

Change to the directory containing the EKS_Fargate configuration:

```bash
cd kubernetes/EKS_Fargate/kubectl
```

Populate the IAM role ARN in [serviceaccount.yaml](kubectl/serviceaccount.yaml):
```yaml
 annotations:
   eks.amazonaws.com/role-arn: <HERE>
```

Next, populate the ACM certificate ARN in [hyperglance.yaml](kubectl/hyperglance.yaml):

```bash
alb.ingress.kubernetes.io/certificate-arn: '<ACM Certificate ARN>' # [REQUIRED] 
# Set to the ARN (e.g. arn:aws:acm:us-east-1:123456789012:certificate/ee760922-32ed-43f9-a46e-4a446b3535c5) of the certificate in ACM you wish to use for the load balancer
```

Next, populate the following values in [volumes.yaml](kubectl/volumes.yaml) with the two EFS filesystem and access point ID you created earlier

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hyperglance-data
  labels:
    app: hyperglance
spec:
   capacity:
      storage: 20Gi
   volumeMode: Filesystem
   accessModes:
      - ReadWriteOnce
   persistentVolumeReclaimPolicy: Retain
   storageClassName: efs-sc
   claimRef:
      name: hyperglance-data
      namespace: hyperglance
   csi:
      driver: efs.csi.aws.com
      volumeHandle: <EFS_Filesystem_ID>::<EFS_Accesspoint_ID_1> # [REQUIRED] Set these values for your deployment fs-075969c3793a0ce8d::fsap-0d583b9784be5fb4c
```

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-data
  labels:
    app: hyperglance
spec:
   capacity:
      storage: 20Gi
   volumeMode: Filesystem
   accessModes:
      - ReadWriteOnce
   persistentVolumeReclaimPolicy: Retain
   storageClassName: efs-sc
   csi:
      driver: efs.csi.aws.com
      volumeHandle: <EFS_Filesystem_ID>::<EFS_Accesspoint_ID_2> # [REQUIRED] Set these values for your deployment e.g. fs-075969c3793a0ce8d::fsap-07f5c973018d10062
```

Deploy the PersistentVolumes, PersistentVolumeClaims and ServiceAccount:

```bash
kubectl apply -f volumes.yaml -f serviceaccount.yaml -n hyperglance
```
*Allow 1-2 minutes before proceeding to the next step*

Finally, deploy Hyperglance:

```bash
kubectl apply -f hyperglance.yaml -n hyperglance
```
*Allow up to 5 minutes before proceeding to the next step*

Now, query the Kubernetes API to find your ALB address and access Hyperglance using this address:

```bash
kubectl get ingress hyperglance -n hyperglance
```

```bash
NAME          CLASS    HOSTS   ADDRESS                                                              
hyperglance   <none>   *       <your_hostname>
```

## Memory management - [guide](https://support.hyperglance.com/knowledge/memory-usage)

If you exerience any OOM (out-of-memory) errors or are scaling up the container resources, you can adjust the MAX_HEAPSIZE values:

Helm -  [values.yaml](helm/hyperglance/values.yaml)

```yaml
MAX_HEAPSIZE: '4096m' # Wildfly heap size setting - adjust accordingly
```

Kubectl - [hyperglance.yaml](kubectl/hyperglance.yaml)

```yaml
MAX_HEAPSIZE: '4096m'  # Wildfly heap size setting - adjust accordingly
```

## Login

Login using the hostname of the ingress ALB. 

Username: admin
Password: admin

It is highly recommended you [change the password](https://support.hyperglance.com/knowledge/how-to-change-hyperglance-login-password) once you login.

Upon login, you will need to go into settings and add your license.
After that, go ahead and add your first AWS account. You can find a guide [here](https://support.hyperglance.com/knowledge/adding-new-aws-accounts-to-hyperglance#first_account_running_in_aws)

:information_source: You will need to define the Role ARN that you created earlier when adding your AWS account to Hyperglance. This is due to the differences in setup between our Marketplace instances and K8s.
