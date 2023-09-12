# BottleRocket + OpenSearch Demo!

Pre-requisites
```bash
brew install eksctl, kubectl, awscli, yq
```

Deploys a cluster based around the `bottlerocket-quickstart-eks.yaml` config. This deploys 5 m5.2xlarge nodes using the bottle rocket AMI and CloudFormation templates.
```bash
eksctl create cluster -f bottlerocket-quickstart-eks.yaml
```

## Ingress Controller
Create the policy for EKS to be able to create application load balanacers. 
```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Quick stript to get the caller identity for use in the next step.
```bash
export DEMOACCOUNT=$(aws sts get-caller-identity | yq e '.Account' -)
```

Create a service account that will allow EKS to spin up load balancers. 
```bash
eksctl create iamserviceaccount \
 --cluster=bottlerocket-opensearch \
 --namespace=kube-system \
 --name=aws-load-balancer-controller-beta \
 --role-name "AmazonEKSLoadBalancerControllerRole" \
 --attach-policy-arn=arn:aws:iam::$(echo $DEMOACCOUNT):policy/AWSLoadBalancerControllerIAMPolicy \
 --region=us-east-2 \
 --override-existing-serviceaccounts \
 --approve
```

Associate the identity provider. 
```bash
eksctl utils associate-iam-oidc-provider \
 --region=us-east-2 \
 --cluster=bottlerocket-opensearch \
 --approve
```

Add the EKS charts for the load balancers. 
```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
 --set clusterName=bottlerocket-opensearch \
 --set serviceAccount.create=false \
 --set serviceAccount.name=aws-load-balancer-controller-beta
```

Validate they are deployed
```bash
kubectl describe deploy aws-load-balancer-controller
```
## Installing OpenSearch
Add and install the OpenSearch Operator.
```bash
helm repo add opensearch-operator https://opster.github.io/opensearch-k8s-operator/
helm install opensearch-operator opensearch-operator/opensearch-operator
```

Deploy the cluster according to the config. 
```bash
kubectl apply -f opensearch-cluster.yaml
```

Wait for all 3 OpenSearch nodes and 1 OpenSearch dashboard nodes to be ready. 
```bash
watch -n 2 kubectl get pods
```

Create ingress for the dashboards. 
```bash
kubectl apply -f dashboards-ingress.yaml
```

Add opensearch username/pw for Fluent-Bit to consume
```bash
kubectl create secret generic opensearchpass \
--from-literal=username=admin \
--from-literal=password=admin
```

Apply the fluentbit operator
```bash
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-operator/release-2.2/manifests/setup/setup.yaml
```

Deploy the fluentbit daemonset
```bash
kubectl apply -f fluentbit-daemonset.yaml 
```

Find the ingress URL for OpenSearch
```bash
kubectl get ingress/ingress-dashboards -n default -o yaml | yq e '.status.loadBalancer.ingress[0].hostname' -
```


Re-Auth
```bash
eksctl utils write-kubeconfig -f bottlerocket-quickstart-eks.yaml
```