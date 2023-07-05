##################### Create AWS EKS clsuster ################################################################################
## Create EKS cluster
eksctl create cluster --name elktest --node-type t2.large --nodes 1 --nodes-min 1 --nodes-max 2 --region us-east-1 --zones=us-east-1a,us-east-1b,us-east-1c

## Get EKS Cluster service
eksctl get cluster --name elktest --region ap-south-1

## Update Kubeconfig 
aws eks update-kubeconfig --name elktest

## Get EKS Pod data.
kubectl get pods --all-namespaces

## Delete EKS cluster
eksctl delete cluster --name oseks --region us-east-1

###############################################################################################################################
##################### Commands to follows #####################################################################################
## CONFIGURE IRSA FOR FLUENT BIT
-----------------------------------
*Enabling IAM roles for service accounts on your cluster
eksctl utils associate-iam-oidc-provider --cluster elktest --approve

*Creating an IAM role and policy for your service account
aws iam create-policy --policy-name fluent-bit-policy --policy-document file://fluent-bit-policy.json

* Create an IAM role

kubectl create namespace logging

eksctl create iamserviceaccount --name fluent-bit --namespace logging --cluster elktest --attach-policy-arn "arn:aws:iam::357171621133:policy/fluent-bit-policy" --approve --override-existing-serviceaccounts

*Make sure your service account with the ARN of the IAM role is annotated
kubectl -n logging describe sa fluent-bit


##PROVISION AN AMAZON OPENSEARCH CLUSTER
------------------------------------------
# name of our Amazon OpenSearch cluster
ES_DOMAIN_NAME="elktest-logging"

# Elasticsearch version
ES_VERSION="OpenSearch_1.0"

# OpenSearch Dashboards admin user
ES_DOMAIN_USER="elktest"

# OpenSearch Dashboards admin password
ES_DOMAIN_PASSWORD="MysuperSct_Ek1$"

*Create the cluster
aws opensearch create-domain --cli-input-json  file://es_domain.json

*Check health state of es_domain
aws opensearch describe-domain --domain-name elktest-logging

## CONFIGURE AMAZON OPENSEARCH ACCESS
----------------------------------------
* We need to retrieve the Fluent Bit Role ARN
eksctl get iamserviceaccount --cluster elktest --namespace logging -o json

* Get the Amazon OpenSearch Endpoint
aws opensearch describe-domain --domain-name elktest-logging --output text --query "DomainStatus.Endpoint"

* Update the Elasticsearch internal database
curl -sS -u "elktest:MysuperSct_Ek1$" -X PATCH https://<OPENSEARCH-END-POINT-FROPM-ABOVE-COMMAND> -H 'Content-Type: application/json' -d '[{"op": "add", "path": "/backend_roles", "value": ["'<FLUENT-BIT-ROLE-ARN>'"]}]'

## DEPLOY FLUENT BIT
----------------------
*Deploy fluentbit on eks
# In fluentbit.yaml file update the opensearch endpoint and aws region accordingly and apply it.

kubectl apply -f ./fluentbit.yaml

*Check status of the pods
kubectl --namespace=logging get pods


## Log in OPENSEARCH
---------------------
*Finally Let’s log into OpenSearch Dashboards to visualize our logs.
OpenSearch Dashboards URL: get this link from AWS console
OpenSearch Dashboards user: your user accordingly
OpenSearch Dashboards password: your password accordingly

################################################################################################################################
