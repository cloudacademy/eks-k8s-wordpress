# eks-k8s-wordpress
EKS K8s Wordpress Deployment

1. Establish RDS secrets

```
RDS_DATABASE_HOSTNAME=database-2.cluster-abcdefg12345.us-west-2.rds.amazonaws.com
kubectl create secret generic mysql-config --from-literal=host=$RDS_DATABASE_HOSTNAME --from-literal=password=password
```

2. Create Cluster OIDC
```
eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --approve
```

3. ALB - Install ALB Ingress Controller
```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json

aws iam create-policy \
 --policy-name AWSLoadBalancerControllerIAMPolicy \
 --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
 --cluster=my_cluster \
 --namespace=kube-system \
 --name=aws-load-balancer-controller \
 --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
 --override-existing-serviceaccounts \
 --approve

kubectl apply \
 --validate=false \
 -f https://github.com/jetstack/cert-manager/releases/download/v1.1.1/cert-manager.yaml

curl -o v2_2_0_full.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/v2_2_0_full.yaml

kubectl apply -f v2_2_0_full.yaml

kubectl get deployment -n kube-system aws-load-balancer-controller
```

4.1 EFS - Create FileSystem

```
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.resourcesVpcConfig.vpcId" --output text)
CIDR_BLOCK=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[].CidrBlock" --output text)
MOUNT_TARGET_GROUP_NAME="eks-efs-group"
MOUNT_TARGET_GROUP_DESC="NFS access to EFS from EKS worker nodes"
MOUNT_TARGET_GROUP_ID=$(aws ec2 create-security-group --group-name $MOUNT_TARGET_GROUP_NAME --description "$MOUNT_TARGET_GROUP_DESC" --vpc-id $VPC_ID | jq --raw-output '.GroupId')
aws ec2 authorize-security-group-ingress --group-id $MOUNT_TARGET_GROUP_ID --protocol tcp --port 2049 --cidr $CIDR_BLOCK
FILE_SYSTEM_ID=$(aws efs create-file-system | jq --raw-output '.FileSystemId')
aws efs describe-file-systems --file-system-id $FILE_SYSTEM_ID
 
aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --security-groups $MOUNT_TARGET_GROUP_ID --subnet-id K8S-NODE-SUBNET-ID-HERE
aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --security-groups $MOUNT_TARGET_GROUP_ID --subnet-id K8S-NODE-SUBNET-ID-HERE
aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --security-groups $MOUNT_TARGET_GROUP_ID --subnet-id K8S-NODE-SUBNET-ID-HERE

aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
aws efs describe-mount-targets --file-system-id $FILE_SYSTEM_ID | jq --raw-output '.MountTargets[].LifeCycleState'
```

4.2 EFS - Install Driver

```
kubectl apply -k https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/deploy/kubernetes/overlays/stable/kustomization.yaml

curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.2.0/docs/iam-policy-example.json
aws iam create-policy \
 --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
 --policy-document file://iam-policy-example.json

eksctl create iamserviceaccount \
 --name efs-csi-controller-sa
 --namespace kube-system \
 --cluster basic-cluster \
 --attach-policy-arn arn:aws:iam::331058736108:policynEKS_EFS_CSI_Driver_Policy \
 --approve \
 --override-existing-serviceaccounts \
 --region us-west-2

kubectl kustomize "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr?ref=release-1.2" > driver.yaml
kubectl apply -f driver.yaml
```

Diagnostics
 
```
kubectl get pods -o wide
kubectl run multitool --image=praqma/network-multitool
kubectl exec -it multitool -- sh
curl -I http://192.168.23.185/index.php
```

```
kubectl run --rm -it --image=mysql:5.7 --restart=Never mysql-client -- mysql -h $RDS_DATABASE_HOSTNAME m -P 3306 -u admin -p
```

```
kubectl logs wordpress-7df79f95df-f5n62 -c wordpress
kubectl logs wordpress-7df79f95df-f5n62 -c nginx
```

```
kubectl exec -it wordpress-7df79f95df-f5n62 -c nginx -- bash
nginx -t
```

```
kubectl exec -it wordpress-7df79f95df-f5n62 -c wordpress -- bash
```