# Intro

Follow the steps below to install an agent to collect network flow logs from your Kubernetes cluster. These network flow logs will be stored in your Cloud Object Storage (COS) instance. You can then enable Security Advisor Network Insights to analyze your network flow logs to detect and alert you to suspicious network activity. Repeat the installation in each Kubernetes cluster that you want to monitor.

# Prerequisites


- IKS > v1.10 and < v1.16. For versions >=v1.16.x, please install charts available at https://github.com/skydive-project/skydive-operator
- For Windows 10, Before starting with steps mentioned above, activate WSL(windows subsystem for linux) and install [ubuntu shell](https://win10faq.com/install-run-ubuntu-bash-windows-10/)
- yq CLI
  - For MacOS, Windows 10: Install [yq CLI](http://mikefarah.github.io/yq/)
  - For CentOS, Red Hat and Ubuntu : Install yq CLI version 1.15 using below steps:  
    `wget https://github.com/mikefarah/yq/releases/download/1.15.0/yq_linux_amd64`  
    `mv yq_linux_amd64 yq`  
    `chmod +x yq`  
    `mv yq /usr/local/bin/`  
    `yq -V`
- curl binary
  - For CentOS and Red Hat, update curl binary using `yum update -y nss curl libcurl`
- Install [Kubernetes CLI (kubectl)](https://kubernetes.io/docs/tasks/tools/install-kubectl/) v1.10.11 or higher
- Install [Kubernetes Helm (package manager)](https://docs.helm.sh/using_helm/#from-script) v2.9.0 or higher
- Please verify whether helm TLS is enabled before proceeding with installation. It is recommended to [enable TLS](https://github.com/helm/helm/blob/master/docs/tiller_ssl.md) in your helm tiller.  
  **Note**:
  - If you are using workstation to handle installation of analytics components in multiple clusters - and the helm TLS is enabled - please make sure that the TLS configurations in your workstation is current and corresponding to current cluster where you are planing to install these components.

# Steps to run

1. Find out the cluster version using:
   ```sh
   kube_version=$(kubectl version --output json)
   echo $(echo $kube_version |  yq r - serverVersion.major).$(echo $kube_version |  yq r - serverVersion.minor)
   ```
   - If output is greater than `1.10`, then download security-advisor-network-insights.tar from `v1.10+` directory in this repo.    
   **Note**: v1.10 is no more supported since May 15th, 2019
2. Unzip using `tar -xvf security-advisor-network-insights.tar`
3. `cd security-advisor-network-insights`
4. Run `./network-insight-install.sh <cos_region> <cos_api_key>`
   - <cos_region> value is either us-south or eu-gb – the region where your COS is deployed
   - <cos_api_key> is the [api key](https://cloud.ibm.com/docs/services/cloud-object-storage/iam/service-credentials.html#service-credentials) you created to access your COS instance and bucket should have a Writer Role.  
     **Note**:
     - This script first validates if a specific bucket with the naming convention sa.<account_id>.telemetric.<cos_region> exists.
     - Creates a Kuberenetes secrets with the following details: cos_region, cos_api_key, cos_endpoint, iam_endpoint, and cos_bucket_name.
     - Updates network-insights-values.yaml with cluster guid and netmask info.
     - Deploys the network insights helm chart into the cluster.
5. Verify the installation :
   - `helm ls | grep network-insights` should return a helm release named `network-insights` in DEPLOYED state, use `--tls` flag if helm is TLS enabled.
   - `kubectl get pods -n security-advisor-insights | grep network-insights` should return two pods related to `network-insights` in RUNNING state.

**Note**: If you create your COS instance and bucket manually (not via Security Advisor UI), make sure to use the following naming convention for the bucket: sa.<account_id>.telemetric.<cos_region>. Also set up service-to-service [authorization](https://cloud.ibm.com/docs/iam/authorizations.html#serviceauth) in IBM Cloud IAM for Security Advisor to read data from your COS instance. Set the source service to Security Advisor and the target service to your cloud object storage instance with a Reader IAM role.

# Deleting the setup

1. `helm del --purge network-insights` , use `--tls` flag if helm is TLS enabled.
2. `kubectl delete ns security-advisor-insights`

**Note**: Repeat for each cluster where you want to remove the agents that collect network flow logs.

# Troubleshooting

1. If you get an error something like `Error: incompatible versions client and server`, run `helm init --upgrade`
2. If you get an error like : `namespaces security-advisor-insights is forbidden: User system:serviceaccount:kube-system:default cannot get resource namespaces in API group in thenamespace security-advisor-insights`, follow below steps:
   ```kubectl delete deployment tiller-deploy -n kube-system
   kubectl apply -f https://raw.githubusercontent.com/IBM-Cloud/kube-samples/master/rbac/serviceaccount-tiller.yaml
   helm init --service-account tiller
   kubectl get pods -n kube-system -l app=helm
   helm list
   ```
