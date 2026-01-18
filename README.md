End-to-end Kubernetes log monitoring using the EFK stack on Amazon EKS

Logging with EFK stack

Here breaking out the Tech-Stack used here mentioned below

🚀 ElastciSearch 

* Monitoring infrastructure and application for Large Ecosystem 
* Store application & system logs
* Time-series analysis

🚀  Fluentbit 

* Act as a agent Closely monitor the application and node forward log to central place 
* Runs on every server or Kubernetes node (as a DaemonSet)
* Lightweight process compare to logstash 
* Efficient in Kubernetes & containerized environments

🚀  Logging Tools For Kubernetes

* EFK Stack (Elasticsearch, Fluent Bit, Kibana) 
* EFK Stack (Elasticsearch, FluentD, Kibana)
* ELK Stack (Elasticsearch, Logstash, Kibana)
* Promtail + Loki + Grafana (industrial and Enterprise Edition)


🚀 System Update & Common Packages
sudo apt update
sudo apt upgrade -y

# Common tools
sudo apt install -y bash-completion wget git zip unzip curl jq net-tools build-essential ca-certificates apt-transport-https gnupg fontconfig


🚀 Reload bash completion if needed:

* source /etc/bash_completion


🚀 AWS CLI Installation

sudo apt install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

🚀 kubectl Installation
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# If the folder `/etc/apt/keyrings` does not exist, create it:
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl bash-completion

# Enable kubectl auto-completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc


🚀 eksctl Installation

ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl

sudo apt-get install -y bash-completion

# Enable eksctl auto-completion
echo 'source <(eksctl completion bash)' >> ~/.bashrc
echo 'alias e=eksctl' >> ~/.bashrc
echo 'complete -F __start_eksctl e' >> ~/.bashrc

source ~/.bashrc

🚀 Helm Installation

sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm bash-completion

# Enable Helm auto-completion
echo 'source <(helm completion bash)' >> ~/.bashrc
echo 'alias h=helm' >> ~/.bashrc
echo 'complete -F __start_helm h' >> ~/.bashrc

source ~/.bashrc

🚀 AWS CLI Configuration
aws configure
aws configure list

🚀 Clone the Repo

git clone https://github.com/harishnshetty/logging-EFK-AWS-EKS-Project.git
cd logging-EFK-AWS-EKS-Project

🚀 Cluster Setup

eksctl create cluster -f 0-eks-creation-config.yml

Check node distribution:

aws eks update-kubeconfig --name my-cluster --region ap-south-1
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone

🚀 Install the AWS EBS CSI Plugin

kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.44"

🚀 Verify deployment:

kubectl get daemonset ebs-csi-node -n kube-system
kubectl get deploy ebs-csi-controller -n kube-system

🚀 Plugin Components:

ebs-csi-node: DaemonSet on every worker node for attaching/mounting EBS volumes.
ebs-csi-controller: Deployment for volume provisioning and lifecycle management.

📦 Deploying the EFK Stack

kubectl create namespace efk-logging --dry-run=client -o yaml | kubectl apply -f -

helm repo add elastic https://helm.elastic.co
helm repo update

helm upgrade --install elasticsearch elastic/elasticsearch \
  -n efk-logging \
  --set replicas=1 \
  --set persistence.enabled=true \
  --set persistence.labels.enabled=true \
  --set volumeClaimTemplate.storageClassName=gp2 \
  --set resources.requests.memory=2Gi \
  --set resources.requests.cpu=1000m \
  --set resources.limits.memory=2Gi \
  --set resources.limits.cpu=1000m \
  --set esJavaOpts="-Xms1g -Xmx1g" \
  --wait

set context - namespace 

  kubectl config set-context --current --namespace=efk-logging 


  Retrieve Elasticsearch Credentials

  # Username
kubectl get secrets --namespace=efk-logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# Password
kubectl get secrets --namespace=efk-logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d

🚀 Install Kibana

helm upgrade --install kibana elastic/kibana \
  --namespace efk-logging \
  --set service.type=LoadBalancer \
  --set replicaCount=1 \
  --wait

 🚀  Install Fluent Bit
 
  Note: Update the HTTP_Passwd field in 1-fluentbit-values.yml

helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm install fluent-bit fluent/fluent-bit \
  -f 1-fluentbit-values.yml \
  -n efk-logging

  Check services:
  kubectl get svc -n efk-logging

  Access Kibana at http://<load-balancer-dns>:5601

  kubectl describe pod elasticsearch-master-0 | grep -i "controlled by"

  🚀 Deploy a Sample Python App

  kubectl create namespace microservice
kubectl apply -f 2-app.yml -n microservice

🧼 Clean Up
kubectl delete pvc elasticsearch-master-elasticsearch-master-0 -n efk-logging
helm uninstall fluent-bit -n efk-logging
helm uninstall elasticsearch -n efk-logging
helm uninstall kibana -n efk-logging
eksctl delete cluster -f 0-eks-creation-config.yml



  




 

  
