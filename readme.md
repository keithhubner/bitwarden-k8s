# Deploying [Bitwarden](https://github.com/bitwarden) to Kubernetes

## Tools used

- Kubectl
- Civo CLI
- Kubectx
- k9s

The purpose of this guide is to demonstrate how to deploy the self hosted version of Bitwarden on Kubernetes. 

## Setting up environment 

For this guide I will be using Civo Kubernetes, because it's fast, cheap and i'm biased! Using the Civo CLI, we can create a new cluster:

```
civo k3s create --save --merge --wait

```

You will see something like this (you will have a different cluster name):

```
The cluster quiet-hawk (7f7698c3-c746-4980-b002-59120231f069) has been created in 1 min 48 sec
```

Then we can select this context:

```
kubectx quiet-hawk
```

## Initial setup

There are a few things we need to setup before deploying the application. First we need to setup a namespace which will contain all the pods:

```
kubectl create ns bitwarden
```

Then we are going to create a config map to store environment variables, create a new file called bitwarden-cm.yaml:

> You will see some of these values are blank at the moment, we will enter these after running the setup program.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: bw-config
  namespace: bitwarden
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  globalSettings__selfHosted: "true"
  globalSettings__pushRelayBaseUri: "https://push.bitwarden.com"
  globalSettings__baseServiceUri__vault: ""
  globalSettings__sqlServer__connectionString: "Data Source=tcp:mssql,1433;Initial Catalog=vault;Persist Security Info=False;User ID=sa;Password=;MultipleActiveResultSets=False;Connect Timeout=30;Encrypt=True;TrustServerCertificate=True"
  globalSettings__identityServer__certificatePassword: ""
  globalSettings__internalIdentityKey: ""
  globalSettings__oidcIdentityClientKey: ""
  globalSettings__duo__aKey: ""
  globalSettings__installation__id: ""
  globalSettings__installation__key: ""
  globalSettings__yubico__clientId: "REPLACE"
  globalSettings__yubico__key: "REPLACE"
  globalSettings__mail__replyToEmail: ""
  globalSettings__mail__smtp__host: "REPLACE"
  globalSettings__mail__smtp__port: "587"
  globalSettings__mail__smtp__ssl: "false"
  globalSettings__mail__smtp__username: "REPLACE"
  globalSettings__mail__smtp__password: "REPLACE"
  globalSettings__disableUserRegistration: "false"
  globalSettings__hibpApiKey: "REPLACE"
  adminSettings__admins: ""
  SA_PASSWORD: ""
  ACCEPT_EULA: "Y"
  MSSQL_PID: "Express"
  NFS_IP: ""
```

```
kubectl apply -f bitwarden-cm.yaml
```

## Deploying the database server

The database server is MSSQL Express and can be treated as "separate" from the application installation.


Then we can apply the k8s maifest:

```
kubectl apply -f bitwarden-mssql.yaml
```

Once the SQL server is up and running, we need to deploy the NFS server which will store all the data for the application.

> We are using an NFS server to allow multiple pods to have read/write to shared storage.

```
kubectl apply -f nfs-server.yaml 
```

Edit the yaml files with the IP address of the NFS service

```
cd nfsshare/
mkdir bwdata

```

### Creating the directories

We need to create the relevant directories on the NFS server:

```
    cd bwdata
    mkdir "core"
    mkdir "core/attachments"
    mkdir "logs"
    mkdir "logs/admin"
    mkdir "logs/api"
    mkdir "logs/events"
    mkdir "logs/icons"
    mkdir "logs/identity"
    mkdir "logs/mssql"
    mkdir "logs/nginx"
    mkdir "logs/notifications"
    mkdir "logs/sso"
    mkdir "logs/portal"
    mkdir "mssql"
    mkdir "mssql/backups"
    mkdir "mssql/data"
    mkdir "web"
    mkdir "ca-certificates"
```

### Running the setup

```
kubectl apply -f bitwarden-setup.yaml
```

```
dotnet Setup.dll -install 1 -domain "YOUR_DOMAIN_HERE" -letsencrypt "y" -os "lin" -corev "latest" -webv "latest" -dbname "vault" -keyconnectorv "latest"
```

Populate the values in the configmap with the values generated 

Reconfigure nginx config for http

Deploy MSSQL

Deploy issuer (staging cert)

Deploy ingress

Deploy all other services



