# 1. Setup infra

### The following dependencies are required on the deployer host:

- Terraform
- Terragrunt
- kubectl
- helm
- aws-iam-authenticator

### AWS requirements
- At least one AWS account
- awscli configured (see installation instructions) to access your AWS account.
- A route53 hosted zone if you plan to use external-dns or cert-manager but it is not a hard requirement.

### Start a new cluster - EKS
```
 git clone https://github.com/anydaytmn/Sartorius-BioSim-infra.git
 ```
 
### Running Terragrunt command

```
cd vpc
terragrunt init
terragrunt apply
```
```
cd eks 
terragrunt init
terragrunt apply
```
```
cd eks-addinns
terragrunt init
terragrunt apply
```
## Install Argo CD  (Linux install or MacOS) 
#### All those components could be installed using a manifest provided by the Argo Project:
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.0.4/manifests/install.yaml
```
## Install Argo CD CLI
#### To interact with the API Server we need to deploy the CLI: 
```
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.0.4/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argoc


export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`
Login
The initial password is autogenerated with the pod name of the ArgoCD API server:

export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
Using admin as login and the autogenerated password:

argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure
```

## ADD Cluster
```
CONTEXT_NAME=`kubectl config view -o jsonpath='{.current-context}'`
 ./argocd cluster add $CONTEXT_NAME 
```

## Install Replace module
```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### ADD DNS record CNAME and ingress configure

ClusterIssuer kubectl apply -f :
```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: your_email_address_here
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

Ingress kubectl apply -f :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: clout
    ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: nginx
  name: argocd
  namespace: argocd
spec:
  rules:
  - host: $host # add hostname
    http:
      paths:
      - backend:
          service:
            name: argocd-server
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - $host # add hostname
    secretName: letsencrypt
```
