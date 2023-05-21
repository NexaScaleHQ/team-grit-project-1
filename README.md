## Microservices Demo Project. Monitoring with Prometheus & Grafana

* Clone the repository <br>
```https://github.com/GoogleCloudPlatform/microservices-demo.git``` <br>

* CD into the cloned repository on your local machine and create an EKS cluster <br>
```eksctl create cluster``` <br>

* Run the following command to create the resources in the kubernetes-manifests.yaml file <br>
```kubectl apply -f ./release/kubernetes-manifests.yaml``` <br> <br>
![image](https://user-images.githubusercontent.com/31238382/234110703-d6ab5a9e-3bd8-4ae2-a829-6d9d00755006.png)

* Check pods <br>
```kubectl get pods``` <br>
![image](https://user-images.githubusercontent.com/31238382/234110744-29b5b8fc-ffb1-4001-874f-c9819f731b66.png)

* List all the services in the currently active Kubernetes context <br>
```kubectl get service``` <br> 

* Deploy the Metrics Server with the following command <br>
```kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml``` <br> <br>
![image](https://user-images.githubusercontent.com/31238382/234110808-a637790e-3a13-41fb-8d45-27171f45d0df.png)

* Verify that the metrics-server deployment is running the desired number of pods with the following command <br>
```kubectl get deployment metrics-server -n kube-system``` <br>
![image](https://user-images.githubusercontent.com/31238382/234110863-a5f26f80-4fdd-4798-a9b2-c81e9e317d35.png)

* Add prometheus Helm repo <br>
```helm repo add prometheus-community https://prometheus-community.github.io/helm-charts``` <br>

* Add grafana Helm repo <br>
```helm repo add grafana https://grafana.github.io/helm-charts```

* Create a prometheus namespace <br>
```kubectl create namespace prometheus``` <br>

* Upgrade or install the Prometheus chart in the prometheus namespace, and set the storage class for both the Alertmanager and the Prometheus server to gp2 <br>
```
helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```
<br>
![image](https://user-images.githubusercontent.com/31238382/234111062-36cdbad1-af0f-442e-a664-e7999f78695a.png)

* Check if Prometheus components deployed as expected <br>
```kubectl get all -n prometheus``` <br>
![image](https://user-images.githubusercontent.com/31238382/234111145-061c6f71-96b7-4bba-89a4-362051aab054.png)

#### If you notice that the prometheus server and alertmanager are stuck on pending, continue the following <br>

* Create an IAM role and service account for the EBS CSI driver in the kube-system namespace of an EKS cluster, attach the AmazonEBSCSIDriverPolicy IAM policy to the role, and approves the creation of the resources
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster scrumptious-painting-1682361990 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```
<br> <br>
![image](https://user-images.githubusercontent.com/31238382/234111368-31dc38c6-1cd1-4163-863f-719193430647.png)

* You’ll get an error after this, then run the following command to  set the oidc_id variable to the value of the OIDC issuer URL for your EKS cluster.
```oidc_id=$(aws eks describe-cluster --name scrumptious-painting-1682361990 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)```
<br> <br>
![image](https://user-images.githubusercontent.com/31238382/234111290-a1b477ef-0455-47fe-b49b-913d03ebea50.png)

* Create an IAM OIDC identity provider for your cluster with the following command. Replace my-cluster with your own value <br>
```eksctl utils associate-iam-oidc-provider --cluster scrumptious-painting-1682361990 --approve```

* Install the ```aws-ebs-csi-driver add-on```, which is used to provide persistent storage for Kubernetes applications running on EKS
```eksctl create addon --name aws-ebs-csi-driver --cluster scrumptious-painting-1682361990 --service-account-role-arn arn:aws:iam::556298987240:role/AmazonEKS_EBS_CSI_DriverRole --force```
<br> <br>
![image](https://user-images.githubusercontent.com/31238382/234111464-1cb5669c-7893-4b9c-bf96-16fa39ee5a78.png)

* Retrieve information about the status of an add-on on an Amazon EKS cluster <br>
```eksctl get addon --name aws-ebs-csi-driver --cluster scrumptious-painting-1682361990```
<br> <br>
![image](https://user-images.githubusercontent.com/31238382/234111521-1fae72c9-777a-4e2e-b932-29d0b8cb45f2.png)

* List all Kubernetes resources in the "prometheus" namespace. Alertmanager and prometheus server should be in the Running state now <br>
```kubectl get all -n prometheus```
<br> <br>
![image](https://user-images.githubusercontent.com/31238382/234111564-95f8bb7f-637c-4a81-aafb-e9acd5b004ea.png)

* Forward traffic from a local port (9090) to the port 9090 of the Prometheus server pod in the "prometheus" namespace. Go to ```http://localhost:9090``` <br>
```kubectl port-forward -n prometheus prometheus-server-77df547d88-rk425 9090:9090```
<br> <br>
![image](https://user-images.githubusercontent.com/31238382/234111683-096ca9f9-eafa-4967-baa3-a01e03bb2ada.png)

* Create a grafana directory and grafana.yaml file
```
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true
 ```

* Update the url in the grafana file <br>
```http://prometheus-server.prometheus.svc.cluster.local```

* Create a grafana namespace <br>
```kubectl create namespace grafana```

* Install Grafana on the EKS cluster using Helm <br>
```
helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='EKS!sAWSome' \
    --values /Users/rolly/Desktop/Learning/ACMP/microservices-demo/grafana/grafana.yaml \
    --set service.type=LoadBalancer
```

* Run the following command to check if Grafana is deployed properly <br>
```kubectl get all -n grafana```

* You can get Grafana ELB URL using this command. Copy & Paste the value into browser to access Grafana web UI.
```
export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "http://$ELB"
```

* This returnses a url - ```http://a511a298d293d4bbca8ba1078dba53fb-1978344538.us-east-1.elb.amazonaws.com```

* Get the password to login to Grafana
```kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo```
 
* Click '+' button on left panel and select ‘Import’.
* Enter 3119 dashboard id under Grafana.com Dashboard.
* Click ‘Load’.
* Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.
* Click ‘Import
![image](https://user-images.githubusercontent.com/31238382/234111807-1484a8ba-c374-4c3c-826e-24f5e6a3978d.png)
![image](https://user-images.githubusercontent.com/31238382/234111844-906e7419-ef96-4273-80fa-38f4ab8ba2e5.png)
![image](https://user-images.githubusercontent.com/31238382/234111873-424af55d-213d-4988-805e-a34873a9bcec.png)
