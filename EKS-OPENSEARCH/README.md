# AWS Opensearch and EKS

## Create and access the eks cluster 
---
Create AWS EKS cluster 

```bash
eksctl create cluster --name elktest --node-type t2.large --nodes 1 --nodes-min 1 --nodes-max 2 --region us-east-1 --zones=us-east-1a,us-east-1b,us-east-1c  
```

Get EKS Cluster service  

```bash
eksctl get cluster --name elktest --region ap-south-1  
```

Update Kubeconfig  

```bash
aws eks update-kubeconfig --name elktest  
```

Check if we are able to access the cluster  

```bash
kubectl get node
```


## CONFIGURE IRSA FOR FLUENT BIT  
-----------------------------------  
Enabling IAM roles for service accounts on your cluster  
```bash
eksctl utils associate-iam-oidc-provider --cluster elktest --approve  
```

Creating an IAM role and policy for your service account  
```bash
aws iam create-policy --policy-name fluent-bit-policy --policy-document file://fluent-bit-policy.json
```  

Create namespace 
```bash
kubectl create namespace logging  
```

Create IAM role

```bash
eksctl create iamserviceaccount --name fluent-bit --namespace logging --cluster elktest --attach-policy-arn "arn:aws:iam::<AWS_ACCOUNT_ID>:policy/fluent-bit-policy" --approve --override-existing-serviceaccounts  
```

Make sure your service account with the ARN of the IAM role is annotated  
```bash
kubectl -n logging describe sa fluent-bit
```


## PROVISION AN AMAZON OPENSEARCH CLUSTER  
------------------------------------------  
Create environment variable 
```bash
export ES_DOMAIN_NAME="elktest-logging"  
export ES_VERSION="OpenSearch_1.0"  
export ES_DOMAIN_USER="elktest"  
export ES_DOMAIN_PASSWORD="MysuperSct_Ek1$"  

```

Create the cluster  
```bash
aws opensearch create-domain --cli-input-json  file://es_domain.json  
```

Check health state of es_domain  
```bash
aws opensearch describe-domain --domain-name elktest-logging  
```

## CONFIGURE AMAZON OPENSEARCH ACCESS  
----------------------------------------  
We need to retrieve the Fluent Bit Role ARN  
```bash
eksctl get iamserviceaccount --cluster elktest --namespace logging -o json  
```

Get the Amazon OpenSearch Endpoint  
```bash
aws opensearch describe-domain --domain-name elktest-logging --output text --query "DomainStatus.Endpoint"  
```

Update the Elasticsearch internal database  
```bash
curl -sS -u "elktest:MysuperSct_Ek1$" -X PATCH https://<OPENSEARCH-END-POINT-FROPM-ABOVE-COMMAND> -H 'Content-Type: application/json' -d '[{"op": "add", "path": "/backend_roles", "value": ["'<FLUENT-BIT-ROLE-ARN>'"]}]'  
```

## DEPLOY FLUENT BIT  
----------------------  
Deploy fluentbit on eks  
In fluentbit.yaml file update the opensearch endpoint and aws region accordingly and apply it.  
```bash
kubectl apply -f ./fluentbit.yaml  
```

Check status of the pods  
```bash
kubectl --namespace=logging get pods  
```


## Log in OPENSEARCH  
---------------------  
Finally Let’s log into OpenSearch Dashboards to visualize our logs.  
OpenSearch Dashboards URL: <GET THIS LINK FROM AWS CONSOLE>
OpenSearch Dashboards user: <YOUR USER ACCORDINGLY>  
OpenSearch Dashboards password: <YOUR PASSWORD ACCORDINGLY>
