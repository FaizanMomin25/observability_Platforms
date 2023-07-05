Make sure you have K8s Cluster Up and running....
=====================EKS + Prometheus + Grafana=========================

1. Clone the kube-state-metrics repository to a local directory:

git clone git@github.com:kubernetes/kube-state-metrics.git

2. Change directories into the cloned repository:

cd kube-state-metrics	

3. Deploy the project to your Kubernetes cluster:

kubectl apply -f examples/standard

========================================

STEP 2: Prometheus Kubernetes Manifest Files

All the configuration files I mentioned in this guide are hosted on Github. You can clone the repo using the following command.

git clone https://github.com/techiescamp/kubernetes-prometheus


STEP 3: Create a Namespace & ClusterRole

First, we will create a Kubernetes namespace for all our monitoring components. If you don’t create a dedicated namespace, all the Prometheus kubernetes deployment objects get deployed on the default namespace.

Execute the following command to create a new namespace named monitoring.

kubectl create namespace monitoring

STEP 4: Create a file named clusterRole.yaml and copy the following RBAC role.

kubectl create -f clusterRole.yaml


STEP 5: Create a Config Map To Externalize Prometheus Configurations.

A: Create a file called config-map.yaml and copy the file contents from this link –> https://raw.githubusercontent.com/bibinwilson/kubernetes-prometheus/master/config-map.yaml

B:  Execute the following command to create the config map in Kubernetes.

kubectl create -f config-map.yaml


STEP 6: Create a Prometheus Deployment

A: Create a file named prometheus-deployment.yaml.

B: Create a deployment on monitoring namespace using the above file.

kubectl create  -f prometheus-deployment.yaml 

C: Step 3: You can check the created deployment using the following command.

kubectl get deployments --namespace=monitoring

 
STEP 7: Exposing Prometheus as a Service [NodePort & LoadBalancer]

A: Create a file named prometheus-service.yaml and copy the following contents. We will expose Prometheus on all kubernetes node IP’s on port 30000.

B: Create the NodePort service using the following command.

kubectl create -f prometheus-service.yaml --namespace=monitoring

C: Access the Prometheus using this nodeport service using PUB IP of any of the cluster node, make sure respective inbond traffic is allowed.

D: Now if you browse to status --> Targets, you will see all the Kubernetes endpoints connected to Prometheus automatically using service discovery as shown below.



STEP 8: Setup Grafana On Kubernetes

A: Create Grafana config 

kubectl create -f grafana-datasource-config.yaml

B: Create Grafana Deployment

kubectl create -f deployment.yaml

C: Create a service to access the grafana.

kubectl create -f service.yaml

D: Now you can access the grafana using node port with your cluster node PUB IP.

http://<your-node-ip>:32000

USER-NAME: admin
PASSWORD: admin

E: Now you can import pre-build grafana dashboards and enjoy.

===========================================
