This will create a namespace called "wordpress".

Then it will contain two deployments, one of mysql and one of wordpress.

```
kubectl apply -f namespace.yml
```

Deploy MYSQL
```
kubectl create secret generic mysql-pass -n wordpress --from-literal=password=$(pwgen 16 1)
kubectl apply -f mysql/01-persistentVolumeClaim.yml
kubectl apply -f mysql/02-deployment.yml
kubectl apply -f mysql/03-service.yml
```

Note: The service has `ClusterIP: None` which will make this a headless service, DNS will point `db` or `db.wordpress.cluster.svc.local` to the IP of the mysql pod. As such, you SHOULD NOT increase the replica count of this mysql database higher than one (1).

Deploy WORDPRESS
```
kubectl apply -f wordpress/01-persistentVolumeClaim.yml
kubectl apply -f wordpress/02-deployment.yml
kubectl apply -f wordpress/03-service.yml
```