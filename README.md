# k8s-cert-manager-opensearch
Demo of Opensearch deployment in k8s with cert manager and self-signed ca certificate

# In the below demo you will see:
Deployment of Cert Manager in k8s
Deployment of Ingress Controller and Resource
Creation of self signed CA using cert manager
Creation of cert manager certificates signed by selfsigned CA
Deployment of Opensearch/Dashboards using helm

# Technologies used:
AWS Cluster
Route53
Cert-Manager
ingress-nginx
helm

# Set up the cluster:

eksctl create cluster \
--name test-cluster \
--region eu-west-1 \
--nodegroup-name linux-nodes \
--node-type t2.medium \
--nodes 3

# Command to delete cluster (Please note the PVC remains on AWS and will accumulate charges if not deleted)
eksctl delete cluster --name test-cluster


# Install cert-manager

kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.7.0 --set installCRDs=true

# Install Ingress Controller

helm install ing ingress-nginx/ingress-nginx \
  --namespace ingress \
  --version 4.0.1 \
  --values ingress_controller.yaml \
  --create-namespace



#After a couple of seconds below command should display External address in a form of domain, update your route53 records accordingly (with Type A record)

kubectl get svc -n ingress


# Install ingress resource

kubectl apply -f ingress.yaml

kubectl get ing

# Create self signed CA

kubectl apply -f example-1

# Examine the created certificates
kubectl get secret -n cert-manager
kubectl get secret ca-key-pair -o yaml -n cert-manager

# Create certificates from the self signed CA that was just created by Cert Manager

kubectl apply -f example-2

# Examine the created certificates

kubectl get secret

kubectl get secret tls-for-dashboards-key-pair -o yaml

# Deploy Opensearch using helm and the created certificates 

cd charts/opensearch

helm package .

helm install --values=values.yaml opensearch opensearch-1.7.1.tgz

# make sure the 3 pods are running

kubectl get pod

# Deploy Dashboards

cd ../opensearch-dashboards/

helm package .

helm install --values=values.yaml dashboards opensearch-dashboards-1.2.0.tgz

# Make sure the logs indicate that the service is running on https://0.0.0.0:5601

k logs dashboards-opensearch-dashboards

# Now connect to your domain in browser (Recommend to use Firefox as the certificates are not trusted by browser, as they are signed by self-signed CA unknown to browser)

test-domain.co.uk
