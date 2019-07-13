This will create a namespace called "wordpress".

Then it will contain two deployments, one of mysql and one of wordpress.

```
kubectl apply -f namespace.yml
```

Edit the following files and replace `my-gcp-project` with the project that contains the DNS for the website name that this wordpress will be deployed on.
NOTE: The assumption here is that you have your DNS in a GCP project and that is is the same as your GKE cluster
- 01-certmanager/08-crd-issuer-production.yaml
- 01-certmanager/07-crd-issuer-staging.yaml
- 03-wordpress/05-certificate.yml

Edit the following files and replace `my-domain.com` with the actual domain that you will be serving wordpress as. Don't make this an IP address as you wont be able to allocate a TLS (read: SSL) Certicate for this via letsencrypt.com

Set some environment variables that we'll use to save you having to edit the commands below
export MY_GCP_PROJECT=my-gcp-project
export MY_DOMAIN=my-domain.com

1. Deploy CertManager (Automated Lets Encrypt Certificate Issuer)
```
kubectl apply -f 01-certmanager/01-namespace.yaml
kubectl apply -f 01-certmanager/02-rbac.yaml
kubectl apply -f 01-certmanager/03-deployment.yaml.erb
kubectl apply -f 01-certmanager/04-crd-certificate.yaml
kubectl apply -f 01-certmanager/05-crd-clusterissuer.yaml
kubectl apply -f 01-certmanager/06-crd-issuer.yaml
```

Create a GCP Service Account in the project which contains your DNS ZONE that you're using
    gcloud beta iam service-accounts create certmanager --description "Automated LetsEncrypt Certificate Management Provisioner" --display-name "Cert Manager (GKE)" --project $MY_GCP_PROJECT
Create a key for this service account and then publish this into your cluster
    gcloud iam service-accounts keys create key.json --project ${MY_GCP_PROJECT} --iam-account cert-manager@${MY_GCP_PROJECT}.iam.gserviceaccount.com

Grant the service account dns administrative access to be able to add/remove dns entries for the purpose of Lets Encrypt domain validation
    gcloud projects add-iam-policy-binding $MY_GCP_PROJECT --member serviceAccount:certmanager${$MY_GCP_PROJECT}.iam.gserviceaccount.com --role roles/dns.admin

kubectl apply -f 01-certmanager/07-crd-issuer-staging.yaml
kubectl apply -f 01-certmanager/08-crd-issuer-production.yaml
```

2. Deploy MYSQL
```
kubectl create secret generic mysql-pass -n wordpress --from-literal=password=$(pwgen 16 1)
kubectl apply -f 02-mysql/01-persistentVolumeClaim.yml
kubectl apply -f 02-mysql/02-deployment.yml
kubectl apply -f 02-mysql/03-service.yml
```

Note: The service has `ClusterIP: None` which will make this a headless service, DNS will point `db` or `db.wordpress.cluster.svc.local` to the IP of the mysql pod. As such, you SHOULD NOT increase the replica count of this mysql database higher than one (1).

3. Deploy WORDPRESS

You will need to reserve an IP address in your GCP project, as this will ensure you don't have any impact of the Ingress is ever deleted and the IP re-allcoated to another customer. The following steps will allocate an ipv4 and ipv6 IP address which is just good practice in 2019.

````
gcloud compute addresses create my-website-v4 --project $MY_GCP_PROJECT --global
gcloud compute addresses create my-website-v6 --project $MY_GCP_PROJECT --global --ip-version IPV6
```

```
kubectl apply -f 03-wordpress/01-persistentVolumeClaim.yml
kubectl apply -f 03-wordpress/02-deployment.yml
kubectl apply -f 03-wordpress/03-backendconfig.yml
kubectl apply -f 03-wordpress/04-service.yml
kubectl apply -f 03-wordpress/05-certificate.yml
kubectl apply -f 03-wordpress/06-ingress.yml
```