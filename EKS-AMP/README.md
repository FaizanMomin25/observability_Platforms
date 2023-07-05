# AWS MANAGED PROMETHEUS WITH GRAFANA ON EKS 
---
---

Make sure you have eks cluster ready with EBS CSI Driver enabled.

### STEP 1 : CREATE AMP WORKSPACE

```bash 
aws amp create-workspace --alias cdex --region ap-south-1
```

----------------------------------------------------------------

### STEP 2 : Setup IAM for Prometheus Server to send metrics to AMP

The shell script shown below can be used to execute the following actions after substituting the placeholder variable YOUR_EKS_CLUSTER_NAME with the name of your Amazon EKS cluster.

* Creates an IAM role with an IAM policy that has permissions to remote-write into an AMP workspace
* Creates a Kubernetes service account that is annotated with the IAM role
* Creates a trust relationship between the IAM role and the OIDC provider hosted in your Amazon EKS cluster
```bash
bash initial-role-setup.sh 
```

----------------------------------------------------------------

### STEP 3 : INSTALL PROMETHEUS SERVER AND INGEST METRICS INTO AMP ( Amazon Managed Prometheus )

A): Execute the following commands to deploy the Prometheus server on the EKS cluster

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

B): Create namespace  

```bash
kubectl create ns prometheus
```

C): As we have to overwrite values.yml file with some configuration changes. So these details are mentioned in **amp_ingest_override_values.yaml**

D): Execute the following command to install the Prometheus server configuration and configure the remoteWrite endpoint

```bash
export SERVICE_ACCOUNT_IAM_ROLE=EKS-AMP-ServiceAccount-Role
export SERVICE_ACCOUNT_IAM_ROLE_ARN=$(aws iam get-role --role-name $SERVICE_ACCOUNT_IAM_ROLE --query 'Role.Arn' --output text)
export WORKSPACE_ID=$(aws amp list-workspaces --alias cdex | jq .workspaces[0].workspaceId -r)
export AWS_REGION=<Your-Region>
```

```bash
helm install prometheus-for-amp prometheus-community/prometheus -n prometheus -f ./amp_ingest_override_values.yaml \
--set serviceAccounts.server.annotations."eks\.amazonaws\.com/role-arn"="${SERVICE_ACCOUNT_IAM_ROLE_ARN}" \
--set server.remoteWrite[0].url="https://aps-workspaces.${AWS_REGION}.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write" \
--set server.remoteWrite[0].sigv4.region=${AWS_REGION}
```

----------------------------------------------------------------

### STEP 4 : INSTALL GRAFANA USING HELM CHARTS

A): Execute the below command to add helm repo for grafana.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

B): As we have to overwrite values.yml file with some configuration changes. So these details are mentioned in **amp_query_override_values.yaml**

*Make sure you change account number and service account role in this file accordingly.*

C): Install grafana.

```bash
helm install grafana-for-amp grafana/grafana -f ./amp_query_override_values.yaml
```

D): Port forwarding for accessing grafana from local machine

```bash
kubectl port-farward pods/<GRAFANA POD NAME> 5001:3000
```

E): You need user name and password for login.

user name : admin

For password run below command in your terminal..

```bash
kubectl get secrets -n grafana grafana-for-amp -o jsonpath='{.data.admin-password}' | base64 --decode
```

Password : output of above command.

----------------------------------------------------------------

### STEP 5 : ADDING AMP AS DATASOURCE IN GRAFANA

A):  Go to AWS Console > Search (Amazon Managed Service for Prometheus) > workspace > Endpoint - query URL(Exclude /api/v1/query part and copy the url )

B): Go to Grafana UI > Home > Administration > data sources > add new data source > select (PROMETHEUS) > add AMP URL from step A >
turn on SigV4 auth > Add default Region As your AWS Region > Save & Test.

Now you can import diffrent dashboard matrices and explore the loging.

##  ***HAPPY MONITORING !!!***
