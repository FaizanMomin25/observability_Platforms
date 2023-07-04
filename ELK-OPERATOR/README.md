============ECK-STACK-WITH-ECK-OPERATOR=========

STEP - 1 :- CREATING EKS CLUSTER

CONFIG FILE :---

vim eks-elk-cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: elk
  region: ap-south-1
nodeGroups:
  - name: worker-nodegroup
    instanceType: t2.large
    desiredCapacity: 3
    ssh:
      publicPath: ~/.ssh/id_rsa.pub
---

eksctl create cluster -f eks-elk-cluster.yaml  
================================================	 

STEP - 2 :- CHECKING OIDC PROVIDER 

● To create an IAM OIDC identity provider for your cluster with eksctl


1. Determine whether you have an existing IAM OIDC provider for your cluster.
Retrieve your cluster's OIDC provider ID and store it in a variable.

oidc_id=$(aws eks describe-cluster --name elk --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

2. Determine whether an IAM OIDC provider with your cluster's ID is already in your
account.

aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4


3. If output is returned, then you already have an IAM OIDC provider for your
cluster and you can skip the next step. If no output is returned, then you must
create an IAM OIDC provider for your cluster.

4. Create an IAM OIDC identity provider for your cluster with the following command. Replace eks-cube-dev with your own value.

eksctl utils associate-iam-oidc-provider --cluster elk --approve

5. Check again whether an IAM OIDC provider with your cluster's ID is created in your account.

aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
-
7D6AF20353E0BA1D8627E65F6E1E81EA”

================================================

STEP - 3 To create your Amazon EBS CSI plugin IAM role with the AWS CLI

1. View your cluster's OIDC provider URL. Replace my-cluster with your cluster name. If the output from the command is None, review the Prerequisites.

aws eks describe-cluster --name elk --query "cluster.identity.oidc.issuer" --output text

https://oidc.eks.ap-south-1.amazonaws.com/id/7D6AF20353E0BA1D8627E65F6E1E81EA


2. Create the IAM role, granting the AssumeRoleWithWebIdentity action.

a. Copy the following contents to a file that's named aws-ebs-csi-driver-trust-policy.json. Replace 111122223333 with your account ID. Replace EXAMPLED539D4633E53DE1B71EXAMPLE and region-code with the values returned in the previous step. If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace arn:aws: with arn:aws-us-gov:


{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::100472621715:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/7D6AF20353E0BA1D8627E65F6E1E81EA"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/7D6AF20353E0BA1D8627E65F6E1E81EA:aud": "sts.amazonaws.com",
          "oidc.eks.ap-south-1.amazonaws.com/id/7D6AF20353E0BA1D8627E65F6E1E81EA:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}

===============

b. Create the role. You can change AmazonEKS_EBS_CSI_DriverRole to a different name. If you change it, make sure to change it in later steps.

aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file://"aws-ebs-csi-driver-trust-policy.json"

==============

3. Attach the required AWS managed policy to the role with the following command. If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace arn:aws: with arn:aws-us-gov:.

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --role-name AmazonEKS_EBS_CSI_DriverRole

==============

STEP - 4 Managing the Amazon EBS CSI driver as an Amazon EKS add-on

Prerequisites

●  An existing cluster. To see the required platform version, run the following command.

aws eks describe-addon-versions --addon-name aws-ebs-csi-driver


● To add the Amazon EBS CSI add-on using the AWS CLI
https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html

aws eks create-addon --cluster-name elk --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::100472621715:role/AmazonEKS_EBS_CSI_DriverRole


OUTPUT :- 
======
{
    "addon": {
        "addonName": "aws-ebs-csi-driver",
        "clusterName": "elk",
        "status": "CREATING",
        "addonVersion": "v1.20.0-eksbuild.1",
        "health": {
            "issues": []
        },
        "addonArn": "arn:aws:eks:ap-south-1:100472621715:addon/elk/aws-ebs-csi-driver/e4c48d20-cf70-e8c1-cff4-b03873e77a64",
        "createdAt": "2023-07-03T11:07:35.630000+05:30",
        "modifiedAt": "2023-07-03T11:07:35.649000+05:30",
        "serviceAccountRoleArn": "arn:aws:iam::100472621715:role/AmazonEKS_EBS_CSI_DriverRole",
        "tags": {}
    }
}
==============

STEP - 5 Install ECK using the helm chart

# helm repo add elastic https://helm.elastic.co
# helm repo update

==============

Cluster-wide (global) installationedit
This is the default mode of installation and is equivalent to installing ECK using the stand-alone YAML manifests.

# helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
==========
NAME: elastic-operator
LAST DEPLOYED: Mon Jul  3 11:19:26 2023
NAMESPACE: elastic-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Inspect the operator logs by running the following command:
   kubectl logs -n elastic-system sts/elastic-operator
===============   

STEP - 6 Deploying The kube-state-metrics on eks cluster

1. Clone the kube-state-metrics repository to a local directory:

# git clone git@github.com:kubernetes/kube-state-metrics.git

2. Change directories into the cloned repository:

# cd kube-state-metrics	

3. Deploy the project to your Kubernetes cluster:

# kubectl apply -f examples/standard

OUTPUT:-- 

###The results should be like the following:

clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
serviceaccount/kube-state-metrics created
service/kube-state-metrics created

4. To test that the KSM service is running, run the following command to expose the service:

# kubectl port-forward svc/kube-state-metrics -n kube-system 8080:8080


5. Now, open up a web browser to address localhost:8080/metrics. If things are working properly, you will see a flow of metrics data from your cluster.

===============

STEP - 7  DEPLOYING  ECK STACK

1. Clone the repo 
 
# git clone https://github.com/elastic/cloud-on-k8s.git

2. We need to disbaled the SSL/TLS setting as we dont have any certificates.

a. cd deploy/eck-stack/charts/eck-elasticserach

b. Here in values.yaml  make the change accordingly shown below.
http: 
  # service:
  #   metadata:
  #     labels:
  #       my-custom: label
  #   spec:
  #     type: LoadBalancer
    tls:
      selfSignedCertificate:
  #     # To fully disable TLS for the HTTP layer of Elasticsearch, simply
  #     # set the below field to 'true', removing all other fields.
        disabled: true
  #     subjectAltNames:
  #       - ip: 1.2.3.4
  #       - dns: hulk.example.com
  #   certificate:
  #     secretName: custom-ca


c. We dont have enterprise license for elasticserach and as ECK-Operator have basic license we need to change annotations for elasticserach as ( cloud-on-k8s/deploy/eck-stack/charts/eck-elasticsearch/templates/elasticserach.yaml)

annotations:
    eck.k8s.elastic.co/license: basic 
	
d. Same as we need to change in ( cloud-on-k8s/deploy/eck-stack/charts/eck-kibana/kibana.yaml	

annotations:
    eck.k8s.elastic.co/license: basic
	
d. Save the changes and install the helm chart going inside the (deploy/eck-stack) folder.

helm install eck-stack .

===============

STEP - 8 Checking for elastic search 	

a. passwd for elasticsearch user: ( username is :- [elastic] ) 

kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode

output :-- 
6u5aw2M44gpp239c9Ii5dhIT

b. k port-forward svc/elasticsearch-es-http 9200

c. Access the elasticsearch on local machine 

localhost:9200
===============

STEP - 9 Accessing Kibana

USERNAME:- elastic

kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
 
PASS:- 6u5aw2M44gpp239c9Ii5dhIT :--- OUTPUT OF ABOVE CMD 


a. k port-forward svc/eck-stack-eck-kibana-kb-http 5601

b. Access the Kibana on local machine

localhost:5601

===============

STEP - 10 Add integration of kubernetes with no changes.

STEP - 11 DOWNLAOD STAND ALONE AGENT MANIFEST FILE

STEP - 12 ADD ELASTIC SEARCH HOST AND PASSWORD IN ABOVE MANIFEST FILE AND APPLY.

Now you should see K8s logs,Metrices in Kibana./..

===============================================================================================
