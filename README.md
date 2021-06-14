# eks-k8s-wordpress
EKS K8s Wordpress Deployment

```
aws ec2 create-volume --region us-west-2 --availability-zone us-west-2a --size 5 --volume-type gp2
```

```
RDS_DATABASE_HOSTNAME=database-2.cluster-abcdefg12345.us-west-2.rds.amazonaws.com
kubectl create secret generic mysql-config --from-literal=host=$RDS_DATABASE_HOSTNAME --from-literal=password=password
```

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