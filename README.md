# Rancher_NeuVector
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

# Stage one - Primary preparation.

## 1. Deploy a SUSE Rancher cluster and through the Rancher GUI, generate a Bearer Token

for single server cluster
```bash
curl -sfL https://get.k3s.io | sh -s - server
for HA cluster:
```
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.23.8+k3s2 sh -s - server --cluster-init
join server to HA cluster
```
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server --server https://<ip or hostname of server1>:6443
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

## 4. Подготовка шаблоных политик для "безопасного" кластера

- Переходим в документацию к Rancher, нам необходим готовые политики для [service account](https://docs.ranchermanager.rancher.io/reference-guides/rancher-security/rancher-v2.6-hardening-guides/rke1-hardening-guide-with-cis-v1.6-benchmark#configure-default-service-account) и [network](https://docs.ranchermanager.rancher.io/reference-guides/rancher-security/rancher-v2.6-hardening-guides/rke1-hardening-guide-with-cis-v1.6-benchmark#configure-network-policy). Политики нужно сохранить на машине, которая будет иметь доступ ко всем кластерам, в идеале она должна иметь Rancher cli на борту.
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

## 5. Установка NeuVector на ns-k3s-fed, ns-k3s-slave и ns-rke-unsecured

### - Установите все кластера longhorn (без него настройки NeuVector не сохранются) 

Cluster Tools - Longhorn - next - Install 

### - Установите NeuVector на k3s:

Cluster Tools - NeuVector - next

Container Runtime: k3s containerd runtime

![11](https://user-images.githubusercontent.com/61315483/189632895-b0b219a7-ca09-4743-889c-47e0bbe6448f.PNG)

PVC Configuration: longhorn

![2](https://user-images.githubusercontent.com/61315483/189632875-d2760028-fa33-46b7-9d10-78a014be38e4.PNG)

Service Configuration: 

   Manager Service Type: NodePort
    
   Fed Master Service Type: NodePort
    
   Fed Managed Service Type: NodePort
    
   Controller Rest API Service Type: NodePort
    
![3](https://user-images.githubusercontent.com/61315483/189632827-d37db35f-ba99-4206-b1a8-e2b5cb3a7658.PNG)
    
  Install
    
### - Установите NeuVector на rke:

Cluster Tools - NeuVector - next 

Container Runtime: Docker runtime

![1](https://user-images.githubusercontent.com/61315483/189632174-5cc1aeca-6c88-4733-9751-e296fd2ed808.PNG)

PVC Configuration: longhorn

![2](https://user-images.githubusercontent.com/61315483/189632403-75f31225-d947-4aba-b54f-07a2c890b069.PNG)

Service Configuration: 

   Manager Service Type: NodePort
    
   Fed Master Service Type: NodePort
    
   Fed Managed Service Type: NodePort
    
   Controller Rest API Service Type: NodePort  
 
![3](https://user-images.githubusercontent.com/61315483/189632579-29644bda-c7a7-4554-842d-555bd850a1b6.PNG)
 
  Install
    
## 6. Настройка NeuVector на ns-k3s-fed, ns-k3s-slave и ns-rke-unsecured

### Перед настройкой любого кластера включите система автосканирование, при этом может пропасть доступ к кластеру на пару минут

![123421](https://user-images.githubusercontent.com/61315483/190161347-b8ff585c-f091-4b2e-9428-b11d3020c223.png)

### - Настройка ns-k3s-fed

Проходим в вэб интерфейс NeuVector, Settings - Configuration - Cluster - Cluster Name (Переименовываем кластер)

![543254325426](https://user-images.githubusercontent.com/61315483/189681256-6f3fcbb6-10a7-4bb5-b44d-f7039066d524.png)

![3214321](https://user-images.githubusercontent.com/61315483/189681536-6b91da88-2d26-4aa6-b8a2-565bb6b73780.PNG)

Повышаем кластер до федеративного лидера, Аватарка администратора - Multiple Clusters - Promote - Primary Cluster Server - Primary Cluster Port

![545432543](https://user-images.githubusercontent.com/61315483/190148925-16c2b866-7a26-4d77-8c06-2fd888ae062b.png)

Данные для заполнения можно получить из Rancher, в Primary Cluster Server можно использовать ip адрес любой ноды кластера. Primary Cluster Port можно получить из Rancher GUI - ns-k3s-fed - Service Discovery - Services - neuvector-svc-controller-fed-master 

![76437654](https://user-images.githubusercontent.com/61315483/190150078-86c6eb4f-548c-400f-87da-c064e27b6872.png)

![6534654](https://user-images.githubusercontent.com/61315483/190150094-7141422c-93d2-49e0-a4bb-a6ab2e67774f.png)

### - Настройка ns-k3s-slave

#### a. Подключаем к федеративному лидеру

Проходим в вэб интерфейс NeuVector, Settings - Configuration - Cluster - Cluster Name (Переименовываем кластер)

Подключаем кластер до федеративному лидеру, Аватарка администратора - Multiple Clusters - Join - Controller Server - Cluster Port

![52432143](https://user-images.githubusercontent.com/61315483/190155330-41ad217a-887f-48ce-91a6-4499d1122cdf.png)

![321321](https://user-images.githubusercontent.com/61315483/190158228-ce080189-910b-402f-8697-eee3f6f27463.png)

Данные для заполнения можно получить из Rancher, в Cluster Server можно использовать ip адрес любой ноды кластера. Cluster Port можно получить из Rancher GUI - ns-k3s-slave - Service Discovery - Services - neuvector-svc-controller-api

![213124321](https://user-images.githubusercontent.com/61315483/190156282-b3a4b3af-174c-4f1c-8ad2-e11512383936.png)

Token можно получить из интерфейса федеративного лидера. Аватарка администратора - Multiple Clusters - Generate Token - Copy

![6543265254](https://user-images.githubusercontent.com/61315483/190157327-ddcde625-612f-4332-8963-c060a758729f.png)

После того как токен будет внесен в поле TOKEN, поля Primary Cluster Server и Primary Cluster Port будут автоматически заполнены.

#### b. Подключаем репозитории

![42542654325](https://user-images.githubusercontent.com/61315483/190408389-f5774e3f-3948-4a32-b004-14c7681b596b.png)

Локальный репозиторий был сделан на основе [данного документа](https://github.com/dff1980/SAPDI-2022#download-images), это важно, так как NeuVector будет понимать какие уязвимости в Rancher, так же это поможет показать как NeuVector работает с локальный репозиторием. 

![5321432143](https://user-images.githubusercontent.com/61315483/190409428-9bc6e28b-66d1-4923-9ed8-d86df7e991b4.PNG)

Подключаем второй репозиторий из DockerHub, данный репозиторий будет проверен на наличие уязвимостей, благодарю этому NeuVector будет понимать как работать с приложени е Hello-Rancher во время демонстрации. 

![531432432](https://user-images.githubusercontent.com/61315483/190410093-b651849f-36fe-4164-8152-063c062cc756.PNG)

#### c. Настойка Admission Control

Перед начало настройки правил переведите STATUS в состояние Enable, и выберите режим работы Monitor или Protect

![415414](https://user-images.githubusercontent.com/61315483/190412845-34a08ab4-72c2-455f-ba64-6cbf086cc6d6.png)

Нажмите кнопку Add и создайте следущие правила (при создании правил не забывайте нажать зеленый плюс): 

![415414](https://user-images.githubusercontent.com/61315483/190413618-f4de44e7-b280-4291-8278-cf2156e0ddac.png)

Rule 1000: 

comment: Deny install Vulnerable apps

Criterion: CVE names

Operator: contains ALL of

Value: CVE-2021-3712,CVE-2021-42386,CVE-2021-36159,CVE-2022-1473,CVE-2021-42378,CVE-2022-0778,CVE-2021-42385,CVE-2022-28391,CVE-2021-4044,CVE-2021-42380,CVE-2021-42382,CVE-2021-3711,CVE-2021-42381,CVE-2021-42383,CVE-2018-25032,CVE-2021-42379,CVE-2022-37434,CVE-2021-42384



Rule 1001:

comment: install only form local registry

Criterion: Image registry

Operator: is one of 

Value: https://index.docker.io/, https://registry.hub.docker.com/, https://registry-1.docker.io/
