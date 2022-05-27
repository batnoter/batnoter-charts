# BatNoter Helm Charts
This project contains the helm charts for batnoter deployment on kubernetes cluster. This repo can be used as a helm repository for batnoter charts.

The github wokflows present in this repo will release and publish the helm chart on push event. This setup is created by referencing the [charts-repo-actions-demo](https://github.com/helm/charts-repo-actions-demo) by helm.

Helm Repository URL - [https://batnoter.github.io/batnoter-charts](https://batnoter.github.io/batnoter-charts)

Helm Repository Metadata URL - 
[https://batnoter.github.io/batnoter-charts/index.yaml](https://batnoter.github.io/batnoter-charts/index.yaml)

**Note:** Make sure to update the chart version before pushing your changes to `main` branch. Otherwise the workflow will fail because release and tag already exist with current version.

## Install batnoter application using helm
### Before you begin
-   Create a kubernetes cluster.
-   Create a postgres database cluster on cloud platform & if required add the trusted sources to it. Make sure to also add the kubernetes cluster in the trusted sources.
-   Postgres cluster may contain default database created by cloud provider. If you want you can create a new one with required name.
-   Connect to database using pgAdmin and create `batnoter` schema inside the database.
-   Make sure you create the kubernetes cluster and database cluster in the same region to avoid any latency issues.

#### Install ingress controller
```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace --set controller.publishService.enabled=true
```

Check if load balancer become available using below command
```shell
kubectl get svc -o wide nginx-ingress-ingress-nginx-controller -n ingress-nginx
```

**NOTE:** Copy the load balancer's external ip address and point it to A-record using your domain's dns interface.
This needs to be done before installing the application. Verify the updates are available using `nslookup` command.
If not then decrease the TTL of A-record and try `nslookup` again.

#### Install cert-manager
```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --version v1.8.0 --set installCRDs=true
```

Verify cert-manager pods
```shell
kubectl get pods -n cert-manager
```

### Install BatNoter Application

#### Create a namespace for BatNoter application
```shell
kubectl create namespace bn
```

#### Create `regcred` secret to allow pulling docker images from the private registry
```shell
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson -n bn
```

**NOTE:** Replace `<path/to/.docker/config.json>` token with correct path

To verify the secret, run the below command
```shell
kubectl get secret regcred -o jsonpath='{.data}' -n bn
```

#### Create `postgres-secret` to allow connecting to postgres database
```shell
kubectl create secret generic postgres-secret \
    --from-literal=DATABASE_USERNAME=<POSTGRES_USER> \
    --from-literal=DATABASE_PASSWORD=<POSTGRES_PASSWORD> \
    --from-literal=DATABASE_DBNAME=<POSTGRES_DB> \
    --from-literal=DATABASE_HOST=<POSTGRES_PORT> \
    --from-literal=DATABASE_PORT=<POSTGRES_PORT> -n bn
```

**NOTE:** Replace `<POSTGRES_USER>` `<POSTGRES_PASSWORD>` `<POSTGRES_DB>` `<POSTGRES_HOST>` and `<POSTGRES_PORT>` with correct values.

To verify the secret, run the below command
```shell
kubectl get secret postgres-secret -o jsonpath='{.data}' -n bn
```

#### Create `auth-secret` to allow user authentication & authorization
```shell
kubectl create secret generic auth-secret \
  --from-literal=OAUTH2_GITHUB_CLIENTID=<GITHUB_CLIENT_ID> \
  --from-literal=OAUTH2_GITHUB_CLIENTSECRET=<GITHUB_CLIENT_SECRET> \
  --from-literal=OAUTH2_GITHUB_REDIRECTURL=https://batnoter.com/api/v1/oauth2/github/callback \
  --from-literal=APP_SECRETKEY=<APP_JWT_SECRET> \
  --from-literal=APP_CLIENTURL=https://batnoter.com -n bn
```

**NOTE:** Replace `<GITHUB_CLIENT_ID>` `<GITHUB_CLIENT_SECRET>` and `<APP_JWT_SECRET>` with correct values.

To verify the secret, run the below command
```shell
kubectl get secret auth-secret -o jsonpath='{.data}' -n bn
```

#### Install the chart with below command
Use the batnoter helm repository to install the application.
```shell
helm repo add batnoter https://batnoter.github.io/batnoter-charts
helm repo update
helm install batnoter batnoter/batnoter -n bn --create-namespace
```

If, for any reason, you want to install batnoter using local helm charts, then run the below command from root directory.
```shell
helm install batnoter charts/batnoter -n bn --create-namespace
```

#### Verify the installation
Check all the pods are up and running with below command
```shell
kubectl get pods -n bn
```

Check all the services is in active state
```shell
kubectl get svc -n bn -o wide
```

Verify Let’s Encrypt certificate status
```shell
kubectl describe certificate batnoter-tls -n bn
kubectl describe clusterissuer letsencrypt-prod -n bn
kubectl get certificaterequest -n bn -o wide
kubectl get orders -n bn
kubectl get challenges -n bn
```

#### Point name-servers of your website to cloud provider
Make sure to link the name-servers of your cloud service provider to your domain using the interface provided by your domain registrar.
Then route the incoming requests to load balancer by creating A-record (DNS record) with the interface provided by cloud service provider.

**NOTE:** If you were using an email hosting service previously with the domain,
then you may need to create respective dns records for the email service provider inside your cloud provider.
Otherwise, the incoming service won't work. This is because we have changed the name-servers of domain registrar.
with the name-servers of cloud service provider. 
Since the DNS records lives inside name-servers, the old DNS records of domain registrar will no longer used.
So make sure to configure respective dns records after updating name-servers.

You should now be able to access your domain on https with a valid certificate.

### Contribution Guidelines
> Every Contribution Makes a Difference

Read the [Contribution Guidelines](CONTRIBUTING.md) before you contribute.
