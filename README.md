# k8s-cert-manager-opensearch
Demo of Opensearch deployment in k8s with cert manager and self-signed ca certificate

## In the below demo you will see:
 - Deployment of Cert Manager in k8s
 - Deployment of Ingress Controller and Resource
 - Creation of self signed CA using cert manager
 - Creation of cert manager certificates signed by selfsigned CA
 - Deployment of Opensearch/Dashboards using helm
 - Generation of certificates using Let's Encrypt
 - Deployment of Opensearch/Dashboards with Let's Encryt certificates

## Technologies used:
 - EKS Cluster
 - Route53
 - Cert-Manager
 - ingress-nginx
 - helm

## Set up the cluster:
```
eksctl create cluster \
--name test-cluster \
--region eu-west-1 \
--nodegroup-name linux-nodes \
--node-type t2.medium \
--nodes 3
```
N.B. Command to delete cluster (Note the PV remains on AWS and will ***accumulate charges if not deleted***)
- eksctl delete cluster --name test-cluster

---


## 1. Install cert-manager
```
 kubectl create namespace cert-manager
 helm repo add jetstack https://charts.jetstack.io
 helm repo update
 helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.7.0 --set installCRDs=true
```
---

## 2. Install Ingress Controller
```
helm install ing ingress-nginx/ingress-nginx \
  --namespace ingress \
  --version 4.0.1 \
  --values ingress_controller.yaml \
  --create-namespace
```

### Update Route53
After a couple of seconds below command should display External address in a form of domain, update your route53 records accordingly (with Type A record)

```
kubectl get svc -n ingress
```

---

## 3. Install ingress resource
```
kubectl apply -f ingress.yaml

kubectl get ing
```
---

## 4. Create self signed CA

```
kubectl apply -f example-1
```

### Examine the created certificates
```
kubectl get secret -n cert-manager
kubectl get secret ca-key-pair -o yaml -n cert-manager
```
---

## 5. Create certificates from the self signed CA that was just created by Cert Manager

```
kubectl apply -f example-2
```

### Examine the created certificates

```
kubectl get secret
kubectl get secret tls-for-dashboards-key-pair -o yaml
```
---

## 6. Deploy Opensearch using helm and the created certificates 
```
cd charts/opensearch
helm package .
helm install --values=values.yaml opensearch opensearch-1.7.1.tgz
```
### Make sure the 3 pods are running

```
kubectl get pod
```
---

## 7. Deploy Dashboards
```
cd ../opensearch-dashboards/
helm package .
helm install --values=values.yaml dashboards opensearch-dashboards-1.2.0.tgz
```
### Make sure the logs indicate that the service is running on https://0.0.0.0:5601

```
kubectl logs dashboards-opensearch-dashboards
```

### Check the result
Now connect to your domain in browser (Recommend to use Firefox as the certificates are not trusted by browser, as they are signed by self-signed CA unknown to browser)

test-domain.co.uk

## 8. Create Let's encrypt certificates

First clear up any certificates and secrets created from previous steps:
```
helm delete dashboards
helm delete opensearch
kubectl delete -f example-2
kubectl delete -f example-1
```
Delete any secrets, so that only below is left:
```
kubectl get secret

NAME                                       TYPE                                  DATA   AGE
default-token-sdzvh                        kubernetes.io/service-account-token   3      11m
```
Apply the new clusterIssuer and certificates files together with additional secrets needed by Opensearch Dashboards
```
kubectl apply -f lets_encrypt/.
```
## 9. Redeploy Opensearch/Dashboards using helm

Perform steps 6 and 7, however this time using new charts located in lets_encrypt/charts directory.

Notice the differences with tls.crt file in opensearch config and initContainer in Opensearch Dashboards config.




