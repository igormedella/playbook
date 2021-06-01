# EKS

Create a **EKS** cluster with **EKSCTL** using Cluster Autoscaling and other cool stuff...

```bash
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws configure
```

```bash
export REGION=us-east-1
export PROFILE=<MY-PROFILE>
export ID=`aws sts get-caller-identity --query Account --output text --profile $PROFILE`
export ENVMODE=test
export CLUSTER=<MY-CLUSTER>
```

```bash
aws sts get-caller-identity --query Account --output text --profile $PROFILE
```

```bash
eksctl create cluster \
 --name $CLUSTER \
 --tags environment=$ENVMODE \
 --region $REGION \
 --with-oidc \
 --zones $REGION"a",$REGION"b",$REGION"c" \
 --version 1.17 \
 --nodegroup-name $CLUSTER"-ng" \
 --nodes-min 10 \
 --nodes-max 20 \
 --node-volume-size 20 \
 --enable-ssm \
 --managed \
 --spot \
 --instance-types t2.micro \
 --asg-access \
 --external-dns-access \
 --full-ecr-access \
 --appmesh-access \
 --appmesh-preview-access \
 --alb-ingress-access \
 --profile $PROFILE
```

OR

```bash
AWS_PROFILE=$PROFILE eksctl create cluster \
 --name $CLUSTER \
 --tags environment=$ENVMODE \
 --region $REGION \
 --with-oidc \
 --zones $REGION"a",$REGION"b",$REGION"c" \
 --version 1.17 \
 --nodegroup-name $CLUSTER"-ng" \
 --nodes-min 10 \
 --nodes-max 20 \
 --node-volume-size 20 \
 --enable-ssm \
 --managed \
 --spot \
 --instance-types t2.micro \
 --asg-access \
 --external-dns-access \
 --full-ecr-access \
 --appmesh-access \
 --appmesh-preview-access \
 --alb-ingress-access \
 --dry-run > eksctl_create_"$CLUSTER"_template.yaml
```

```bash
nano -c eksctl_create_"$CLUSTER"_template.yaml
```

**Prerequisites:** Auto Scaling Group Tagging

Then the pod needs to know which auto-scaling group to manipulate, so you need a tag on the ASG so that pod knows which ASG belongs to the cluster it lives in. Tagging the ASG with the following key/values:

```text
k8s.io/cluster-autoscaler/<cluster-name>: owned
k8s.io/cluster-autoscaler/enabled: true
```

```bash
eksctl create cluster -f eksctl_create_"$CLUSTER"_template.yaml --profile $PROFILE
```

```bash
aws eks update-kubeconfig --name $CLUSTER --region $REGION --profile $PROFILE
```

```bash
kubectl config delete-context $PROFILE@$CLUSTER.$REGION.eksctl.io
aws eks list-clusters --region $REGION --profile $PROFILE
aws eks --region $REGION update-kubeconfig --name $CLUSTER --profile $PROFILE
kubectl config get-contexts
kubectl config use-context arn:aws:eks:$REGION:$ID:cluster/$CLUSTER
kubectl config current-context
```

## Deploy the Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

```bash
kubectl get deployment metrics-server -n kube-system
```

## Deploy Prometheus using Helm

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```bash
helm install prometheus prometheus-community/prometheus \
--namespace prometheus \
--create-namespace \
--set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

```bash
kubectl get pods -n prometheus
```

### Get the PushGateway URL by running these commands in the same shell

```bash
export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace prometheus port-forward $POD_NAME 9091
```

## Deploy Grafana using Helm

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```bash
helm install grafana stable/grafana \
--namespace grafana \
--create-namespace \
--set grafana.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

OR

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

```bash
helm install grafana bitnami/grafana \
--namespace grafana \
--create-namespace \
--set grafana.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

```bash
kubectl get pods -n grafana
```

### Get the application URL by running these commands

```bash
echo "Browse to http://127.0.0.1:8080"
kubectl port-forward svc/grafana 8080:3000 &
```

### Get the admin credentials

```bash
echo "User: admin"
echo "Password: $(kubectl get secret grafana-admin --namespace grafana -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 --decode)"
```

**Dashboards:** <https://grafana.com/orgs/itewk/dashboards>

## Deploy Kubenav

```bash
kubectl apply --kustomize github.com/kubenav/deploy/kustomize
kubectl port-forward --namespace kubenav svc/kubenav 14122
```

## Expose Services

```bash
kubectl expose deployment prometheus-server --namespace prometheus --type=LoadBalancer --name=prometheus-loadbalancer
kubectl expose deployment grafana --namespace grafana --type=LoadBalancer --name=grafana-loadbalancer
kubectl expose deployment kubenav --namespace kubenav --type=LoadBalancer --name=kubenav-loadbalancer
```

## Show EXTERNAL-IP

```bash
kubectl get service/prometheus-loadbalancer --namespace prometheus |  awk {'print $1" " $2 " " $4 " " $5'} | column -t
kubectl get service/grafana-loadbalancer --namespace grafana |  awk {'print $1" " $2 " " $4 " " $5'} | column -t
kubectl get service/kubenav-loadbalancer --namespace kubenav |  awk {'print $1" " $2 " " $4 " " $5'} | column -t
```

OR

```bash
kubectl get services --all-namespaces | grep loadbalancer | awk {'print $1" " $2 " " $4 " " $5'} | column -t
```

## Enable Auto Scaling

```bash
wget -c https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

```bash
nano -c cluster-autoscaler-autodiscover.yaml
```

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

### Deploy a test application

```bash
kubectl apply -f https://raw.githubusercontent.com/smashse/playbook/master/HOWTO/KUBERNETES/COMBO/example_combo_full.yaml
```

```bash
kubectl get nodes -o wide
```

```bash
kubectl get pods -o=custom-columns=NODE:.spec.nodeName,NAME:.metadata.name --namespace teste
```

```bash
kubectl scale deployment teste-deployment --replicas=10 --namespace teste
```

```bash
kubectl get pods -o=custom-columns=NODE:.spec.nodeName,NAME:.metadata.name --namespace teste
```

**Source:**
<https://eksctl.io/usage/autoscaling/>
<https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md>

## Deploy NGINX

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/aws/deploy.yaml
```

## Delete EKS cluster

```bash
eksctl delete cluster --name $CLUSTER --region $REGION --profile $PROFILE
kubectl config delete-context $PROFILE@$CLUSTER.$REGION.eksctl.io
kubectl config delete-context arn:aws:eks:$REGION:$ID:cluster/$CLUSTER
```

**Source:**

<https://kubernetes.github.io/ingress-nginx/deploy/>
<https://docs.aws.amazon.com/eks/latest/userguide/eks-managing.html>
<https://aws.amazon.com/pt/premiumsupport/knowledge-center/eks-kubernetes-services-cluster/>
<https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights-Prometheus-Sample-Workloads-nginx.html>
<https://github.com/kubenav/kubenav>