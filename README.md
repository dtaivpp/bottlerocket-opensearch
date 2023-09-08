# BottleRocket + OpenSearch Demo!

```bash
brew install eksctl, kubectl, awscli, yq
```

```bash
source .env
```


```bash
eksctl create cluster -f bottlerocket-quickstart.yaml
```

```bash
export DEMOACCOUNT=$(aws sts get-caller-identity —no-cli-pager | yq e '.Account' -)
```

```bash
eksctl utils associate-iam-oidc-provider --region=us-east-2 --cluster=bottlerocket-opensearch-beta --approve
```


```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
 -n kube-system \
 --set clusterName=bottlerocket-opensearch-beta \
 --set serviceAccount.create=false \
 --set serviceAccount.name=bottlerocket-opensearch-beta
```

```bash
eksctl create iamserviceaccount \
 --cluster=bottlerocket-opensearch-beta \
 --namespace=kube-system \
 --name=aws-load-balancer-controller-beta \
 --role-name "AmazonEKSLoadBalancerControllerRole-beta" \
 --attach-policy-arn=arn:aws:iam::$(echo $DEMOACCOUNT):policy/AWSLoadBalancerControllerIAMPolicy \
 --region=us-east-2 \
 --approve

```

```bash
kubectl describe deploy -n kube-system aws-load-balancer-controller
```


```bash
kubectl apply -f ./api-pod-daemonset.yaml
```

```bash
kubectl delete -f  api-pod-daemonset.yaml
```

```bash
helm repo add opensearch-operator https://opster.github.io/opensearch-k8s-operator/
```

```bash
helm install opensearch-operator opensearch-operator/opensearch-operator
```

Re-Auth
```bash
eksctl utils write-kubeconfig -f bottlerocket-quickstart-eks.yaml
```

```bash
kubectl apply -f opensearch-cluster.yaml
```

```bash
watch -n 2 kubectl get pods
```

```bash
kubectl apply -f dashboards-ingress.yaml
```


```bash
kubectl create secret generic opensearchpass \
--from-literal=username=admin \
--from-literal=password=admin
```

```bash
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-operator/release-2.2/manifests/setup/setup.yaml
```