# Rancher+NeuVector
### Project Goal: Achieve Rapid Deployment of Secure K8S Clusters and their subsequent fine-tuning with NeuVector

While working on this project, i used information from this document [Rancher's hardening guide](https://docs.ranchermanager.rancher.io/pages-for-subheaders/rancher-v2.6-hardening-guides).

I used [this document](https://github.com/dff1980/SAPDI-2022#download-images) to create a local repository.

## Versions used
- k3s v1.23.8+k3s2
- Rancher v2.6.7
- SLES 15 SP3
- kubectl latest
- NeuVector 5.0.2
- rke v1.21.8-rancher1-1
- VMware vSphere 6.7  

## 1. Deploy a SUSE Rancher cluster and through the Rancher GUI, generate a Bearer Token

for single server cluster
```bash
curl -sfL https://get.k3s.io | sh -s - server
```
for HA cluster:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.23.8+k3s2 sh -s - server --cluster-init
```
join server to HA cluster
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server --server https://<ip or hostname of server1>:6443
```
join worker nodes to cluster
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<ip or hostname of server1>:6443  K3S_TOKEN=SECRET sh -
```
```bash
helm repo add jetstack https://charts.jetstack.io
```
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```
```bash
helm repo update
```
```bash
helm install cert-manager jetstack/cert-manager \
--create-namespace \
--namespace cert-manager \
--version v1.7.1 \
--set installCRDs=true \
--set nodeSelector."kubernetes\.io/os"=linux \
--set webhook.nodeSelector."kubernetes\.io/os"=linux \
--set cainjector.nodeSelector."kubernetes\.io/os"=linux \
--set startupapicheck.nodeSelector."kubernetes\.io/os"=linux
```  
```bash  
helm install rancher rancher-latest/rancher \
--create-namespace \
--namespace cattle-system \
--set hostname=<IP_OF_LINUX_NODE>.sslip.io \
--set replicas=3 \
--set bootstrapPassword=<PASSWORD_FOR_RANCHER_ADMIN>
```
- Click on your account avatar Rancher - Account & API Keys - Create API Key - Create - Bearer Token 

save Bearer Token, it will be needed to access k8s clusters via rancher cli. 

## 2. Preparing an RKE Template for a "Secure" Cluster

- Let's go to the Rancher documentation, we need [RKE template](https://docs.ranchermanager.rancher.io/reference-guides/rancher-security/rancher-v2.6-hardening-guides/rke1-hardening-guide-with-cis-v1.6-benchmark#reference-hardened-rke-template-configuration) copy it.
- In Rancher GUI we create an RKE template (Cluster Management-RKE1 Configuration-RKE Templates-Add Template-Edit as YAML), paste the template copied in the previous step.

## 3. Preparing a Node Template for a "Secure" Cluster

- Let's go to the documentation for Rancher, we need [template for SLES15](https://docs.ranchermanager.rancher.io/reference-guides/rancher-security/rancher-v2.6-hardening-guides/rke1-hardening-guide-with-cis-v1.6-benchmark#reference-hardened-cloud-config-for-suse-linux-enterprise-server-15-sles-15-and-opensuse-leap-15) copy it.
- Create an RKE node template in the Rancher GUI (Cluster Management-RKE1 Configuration-Node Templates-Add Template), paste the template copied in the previous step into the section Cloud Config YAML. 

## 4. Preparing Policy Templates for a "Secure" Cluster

- We turn to the documentation for Rancher, we need ready-made policies for [service account](https://docs.ranchermanager.rancher.io/reference-guides/rancher-security/rancher-v2.6-hardening-guides/rke1-hardening-guide-with-cis-v1.6-benchmark#configure-default-service-account) and [network](https://docs.ranchermanager.rancher.io/reference-guides/rancher-security/rancher-v2.6-hardening-guides/rke1-hardening-guide-with-cis-v1.6-benchmark#configure-network-policy). Policies need to be saved on a machine that will have access to all clusters, ideally it should have Rancher cli on board.
### a. Service account
```bash
vi account_update.yaml
```
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
automountServiceAccountToken: false
```
```bash
vi account_update.sh
```
```bash
#!/bin/bash -e

for namespace in $(kubectl get namespaces -A -o=jsonpath="{.items[*]['metadata.name']}"); do
  kubectl patch serviceaccount default -n ${namespace} -p "$(cat account_update.yaml)"
done
```
```bash
chmod +x account_update.sh
```

### b. Network Policy
```bash
vi default-allow-all.yaml
```
```bash
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-all
spec:
  podSelector: {}
  ingress:
  - {}
  egress:
  - {}
  policyTypes:
  - Ingress
  - Egress
```
```bash
vi apply_networkPolicy_to_all_ns.sh
```
```bash
#!/bin/bash -e

for namespace in $(kubectl get namespaces -A -o=jsonpath="{.items[*]['metadata.name']}"); do
  kubectl apply -f default-allow-all.yaml -n ${namespace}
done
```
```bash
chmod +x apply_networkPolicy_to_all_ns.sh
```

## 5. Deploy two or more cluster's using RKE template and nodes template

Cluster Management - Cluster - Create - VMware vSphere 

- Use Secured RKE template
- Use Secured NODE template

## 6. Install NeuVector in new secured cluster's

### - Install all longhorn on all clusters (without it, NeuVector settings will not be saved) 

Cluster Tools - Longhorn - next - Install 
    
### - Install NeuVector на rke:

Cluster Tools - NeuVector - next 

Container Runtime: Docker runtime

PVC Configuration: 
   
   Chech: "PVC status"
   
   Storage Class name: longhorn 

Service Configuration: 

   Manager Service Type: NodePort
    
   Fed Master Service Type: NodePort
    
   Fed Managed Service Type: NodePort
    
   Controller Rest API Service Type: NodePort  
 
  Install
    
### Before setting up any cluster, enable the autoscan system, when this setting is enabled, access to the cluster may be lost for a couple of minutes

## 7. Config NeuVector on Primary cluster 

Go to the web interface NeuVector, Settings - Configuration - Cluster - Cluster Name (Rename cluster)

Promoting a Cluster to a Primary Cluster, Administrator Avatar - Multiple Clusters - Promote - Primary Cluster Server - Primary Cluster Port

The data for filling can be obtained from Rancher, in the Primary Cluster Server, you can use the ip address of any cluster node. Primary Cluster Port can be obtained from Rancher GUI - PRIMARY_CLUSTER_NAME - Service Discovery - Services - neuvector-svc-controller-fed-master 

## 8. Config NeuVector on Minion cluster

Go to the web interface NeuVector, Settings - Configuration - Cluster - Cluster Name (Rename cluster)

Connect the minion cluster to the Primary Cluster, Avatar of the administrator - Multiple Clusters - Join - Controller Server - Cluster Port

The data for filling can be obtained from Rancher, in Cluster Server you can use the ip address of any cluster node. Cluster Port can be obtained from Rancher GUI - MINION_CLUSTER_NAME - Service Discovery - Services - neuvector-svc-controller-api

The TOKEN can be obtained from the interface of the federated leader. Administrator avatar - Multiple Clusters - Generate Token - Copy

After the token is entered in the TOKEN field, the Primary Cluster Server and Primary Cluster Port fields will be automatically filled.

## 9 Apply network and pod police

On a machine that has access to the rancher server and carries rancher cli type in:

rancher login https://%rancherurl% --token BearerToken  --skip-verify

select target cluster and press enter

apply policy

```bash
./account_update.sh
```
```bash
apply_networkPolicy_to_all_ns.sh
```
## 10 Install CIS Benchmark and run scan

Cluster Tools - CIS Benchmark - next - Install

After install you can run all Hardened profiles scans

## We have a couple of working clusters with NeuVector installed
