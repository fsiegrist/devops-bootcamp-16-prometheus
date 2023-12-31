## Exercises for Module 16 "Monitoring with Prometheus"
<br />

**Context**

You and your team are running the following setup in the K8s cluster:

Java application that uses Mysql DB and is accessible from browser using Ingress. It's all running fine, but sometimes you have issues where Mysql DB is not accessible or Ingress has some issues and users can't access your Java application. And when this happens, you and your team spend a lot of time figuring out what the issue is and troubleshooting within the cluster. Also, most of the time when these issues happen, you are not even aware of them until an application user writes to the support team that the application isn't working or developers write you an email that things need to be fixed.

As an improvement, you have decided to increase visibility in your cluster to know immediately when such issues happen and proactively fix them. Also, you want a more efficient way to pinpoint the issues right away, without hours of troubleshooting. And maybe even prevent such issues from happening by staying alert to any possible issues and fixing them before they even happen.

Your manager suggested using Prometheus, since it's a well known tool with a large community and is widely used, especially in K8s environment.

So you and your team are super motivated to improve the application observability using Prometheus monitoring.

<details>
<summary>Exercise 1: Deploy your Application and Prepare the Setup</summary>
<br />

**Tasks:**

- Create a K8s cluster
- Deploy Mysql database for your Java application with 2 replicas (You can use the following helm chart: https://github.com/bitnami/charts/tree/master/bitnami/mysql)
- Deploy Java Maven application with 3 replicas that talks to the Mysql DB
- Deploy Nginx Ingress Controller (You can use the following helm chart: https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx)
- Now configure access to your Java application using an Ingress rule

You can use the Ansible playbook from Ansible exercises 7 & 8 with a few adjustments to configure this setup. 

**Solution:**

**Create a K8s cluster on LKE and set the kubeconfig file**\
Login to your [Linode account](https://cloud.linode.com/), press the blue "Create" button and select "Kubernetes". Enter a cluster label (e.g. 'monitoring'), choose a region close to you (e.g. 'Frankfurt, DE (eu-central)') and select the latest Kubernetes version (e.g. 1.26). Check 'No' for high availability control plane. In the "Add Node Pools" section select the "Shared CPU" tab and add 2 "Linode 4 GB" nodes to the cart. Press the "Create Cluster" button.\
On the dashboard you can see the two worker nodes (Linodes). Wait until both are up and running.

In the Kubernetes section at the top you can download a 'monitoring-kubeconfig.yaml' file with the credentials and certificates you need to connect to the K8s cluster. Download it and set the environment variable KUBECONFIG on your local machine to this file:
```sh
chmod 400 ~/Downloads/monitoring-kubeconfig.yaml
export KUBECONFIG=~/Downloads/monitoring-kubeconfig.yaml

# now kubectl commands will be connected with the linode cluster
kubectl get nodes
# NAME                            STATUS   ROLES    AGE   VERSION
# lke132137-194973-0bb2a7520000   Ready    <none>   77s   v1.26.3
# lke132137-194973-5414aa870000   Ready    <none>   41s   v1.26.3
```

**Create a docker-registry secret**\
```sh
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=fsiegrist
DOCKER_EMAIL=<your-dockerhub-email>
DOCKER_PASSWORD=<your-dockerhub-pwd>

kubectl create secret docker-registry my-registry-key\
  --docker-server=$DOCKER_REGISTRY_SERVER\
  --docker-username=$DOCKER_USER\
  --docker-password=$DOCKER_PASSWORD\
  --docker-email=$DOCKER_EMAIL

# secret/my-registry-key created
```

**Build the application and push image to private Docker Registry**
```sh
cd bootcamp-java-mysql
./gradlew build

# build and push the docker image using buildx on an apple M2 machine
docker buildx create --use
docker login
docker buildx build --platform linux/amd64 -t fsiegrist/fesi-repo:bootcamp-java-mysql-monitoring-1.0 --push .

# or conventionally on an amd64 machine
docker build -t fsiegrist/fesi-repo:bootcamp-java-mysql-monitoring-1.0 .
docker push fsiegrist/fesi-repo:bootcamp-java-mysql-monitoring-1.0
```

**Execute Ansible playbook to deploy java and mysql apps in k8s cluster**\
Adjust the value of the `hosts` attribute in 'kubernetes-manifests/java-app-ingress.yaml' to the IP address or reverse DNS of one of the nodes in the LKE cluster. The execute the ansible playbook to configure the cluster:
```sh
ansible-playbook ex1-configure-k8s.yaml
```

After a couple of minutes the mysql and java-app pods should be running:
```sh
kubectl get pods
# NAME                                   READY   STATUS    RESTARTS        AGE
# java-app-deployment-57d9d54dbb-57cxv   1/1     Running   4 (2m17s ago)   3m35s
# java-app-deployment-57d9d54dbb-lhq5c   1/1     Running   4 (2m15s ago)   3m35s
# java-app-deployment-57d9d54dbb-sn949   1/1     Running   5 (96s ago)     3m35s
# mysql-release-primary-0                1/1     Running   0               3m38s
# mysql-release-secondary-0              1/1     Running   0               3m38s
```

**NOTES:**\
If you get an error on creating ingress component related to "nginx-controller-admission" webhook, than manually delete the ValidationWebhook and try again. To delete the ValidationWebhook:
```sh
kubectl get ValidatingWebhookConfiguration # gives you the name
kubectl delete ValidatingWebhookConfiguration {name}
```

</details>

******

<details>
<summary>Exercise 2: Start Monitoring your Applications</summary>
<br />

**Tasks:**

Note: as you've learned, we deploy separate exporter applications for different services to monitor third party applications. But, some cloud native applications may have the metrics scraping configuration inside and not require an addition exporter application. So check whether the chart of that application supports scraping configuration before deploying a separate exporter for it.

- Deploy Prometheus Operator in your cluster (You can use the following helm chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- Configure metrics scraping for Nginx Controller
- Configure metrics scraping for Mysql
- Configure metrics scraping for Java application (Note: Java application exposes metrics on port 8081, NOT on /metrics endpoint)
- Check in Prometheus UI, that all three application metrics are being collected


**Solution:**

**Deploy promentheus operator**
```sh
# make sure the helm charts repo 'prometheus-community' is available
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# create a namespace 'monitoring' and install the prometheus stack in it
kubectl create namespace monitoring
helm install monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring
# NAME: monitoring-stack
# LAST DEPLOYED: Sun Sep 24 15:32:58 2023
# NAMESPACE: monitoring
# STATUS: deployed
# REVISION: 1
# NOTES:
# kube-prometheus-stack has been installed. Check its status by running:
#   kubectl --namespace monitoring get pods -l "release=monitoring-stack"
```

**Access Prometheus UI and view its targets**
```sh
kubectl port-forward svc/monitoring-stack-kube-prom-prometheus 9090:9090 -n monitoring
```
Open the browser and navigate to [http://127.0.0.1:9090/targets](http://127.0.0.1:9090/targets).

**NOTE:**
We are using the label 'release: monitoring-stack' to expose scrape endpoints. This label may change with newer prometheus stack versions, so to check which label you need to apply, do the following:
```sh
# get name of the prometheus CRD
kubectl get prometheus -n monitoring
# NAME                                    VERSION   DESIRED   READY   RECONCILED   AVAILABLE   AGE
# monitoring-stack-kube-prom-prometheus   v2.47.0   1         1       True         True        21m

# print out the ServiceMonitor selector
kubectl get prometheus monitoring-stack-kube-prom-prometheus -n monitoring -o yaml | grep serviceMonitorSelector -A 2
#   serviceMonitorSelector:
#     matchLabels:
#       release: monitoring-stack      <-----
```

**Add metrics scraping to nginx, mysql and java apps**
```sh
ansible-playbook ex2-configure-k8s.yaml
```

Access Prometheus UI and see that new targets for mysql, nginx and your java application have been added. Open the browser and navigate to [http://127.0.0.1:9090/targets](http://127.0.0.1:9090/targets):
- serviceMonitor/default/java-app-sm/0 (3/3 up)
- serviceMonitor/default/mysql-release/0 (2/2 up)
- serviceMonitor/ingress/ingress-controller-ingress-nginx-controller/0 (1/1 up)

</details>

******

<details>
<summary>Exercise 3: Configure Alert Rules</summary>
<br />

**Tasks:**

Now it's time to configure alerts for critical issues that may happen with any of the applications.

- Configure an alert rule for nginx-ingress: More than 5% of HTTP requests have status 4xx
- Configure alert rules for Mysql: All Mysql instances are down & Mysql has too many connections
- Configure alert rule for the Java application: Too many requests
- Configure alert rule for a K8s component: StatefulSet replicas mismatch (Since Mysql is deployed as a StatefulSet, if one of the replicas goes down, we want to be notified)


**Solution:**

Apply the following alert rule configurations to the cluster:
```sh
kubectl apply -f kubernetes-manifests/ex3-nginx-alert-rules.yaml
# prometheusrule.monitoring.coreos.com/ingress-rules created

kubectl apply -f kubernetes-manifests/ex3-mysql-alert-rules.yaml
# prometheusrule.monitoring.coreos.com/mysql-rules created

kubectl apply -f kubernetes-manifests/ex3-java-alert-rules.yaml
# prometheusrule.monitoring.coreos.com/java-rules created

kubectl apply -f kubernetes-manifests/ex3-k8s-alert-rules.yaml
# prometheusrule.monitoring.coreos.com/k8s-rules created
```

Access Prometheus UI and see that 5 new alert rules have been added (one for java, k8s and ingress; two for mysql). Open the browser and navigate to [http://127.0.0.1:9090/alerts](http://127.0.0.1:9090/alerts) or [http://127.0.0.1:9090/rules](http://127.0.0.1:9090/rules).

</details>

******

<details>
<summary>Exercise 4: Send Alert Notifications</summary>
<br />

**Tasks:**

Great job! You have added observability to your cluster, and you have configured your monitoring with all the important alerts. Now when issues happen in the cluster, you want to automatically notify people who are responsible for fixing the issue or at least observing the issue, so it doesn't break the cluster.

- Configure alert manager to send all issues related to Java or Mysql application to the developer team's Slack channel. (Hint: You can use the following guide to set up a Slack channel for the notifications: https://www.freecodecamp.org/news/what-are-github-actions-and-how-can-you-automate-tests-and-slack-notifications/#part-2-post-new-pull-requests-to-slack)
- Configure alert manager to send all issues related Nginx Ingress Controller or K8s components to K8s administrator's email address.

Note: Of course, in your case, this can be your own email address or your own Slack channel.


**Solution:**

Use this [freecodecamp](https://www.freecodecamp.org/news/what-are-github-actions-and-how-can-you-automate-tests-and-slack-notifications/) guide to set up your Slack channel.

Configure your email account as described in the [demo project #2](../demo-projects/2-alerting/). Copy the app-password value and write it into the `kubernetes-manifests/ex4-email-secret.yaml` Secret configuration (base64 encoded).

Execute the following commands to configure the alert manager to send notifications:
```sh
kubectl apply -f kubernetes-manifests/ex4-email-secret.yaml
# secret/gmail-auth created

kubectl apply -f kubernetes-manifests/ex4-slack-secret.yaml
# secret/slack-auth created

kubectl apply -f kubernetes-manifests/ex4-alert-manager-configuration.yaml
# alertmanagerconfig.monitoring.coreos.com/main-rules-alert-config created
```

Create a port-forwarding for the Alertmanager UI:
```sh
kubectl port-forward svc/monitoring-stack-kube-prom-alertmanager -n monitoring 9093:9093
```

Open the browser and navigate to [http://localhost:9093/#/status](http://localhost:9093/#/status) to see the added routes and receivers in the configuration of the Alertmanager.

</details>

******

<details>
<summary>Exercise 5: Test the Alerts</summary>
<br />

**Tasks:**

Of course, you want to check now that your whole setup works, so try to simulate issues and trigger 1 alert for each notification channel (Slack and E-mail).

For this, you can simply kubectl delete one of the stateful set pods, or Mysql pods or try accessing your java applications on a /path-that-doesnt-exist etc. 


**Solution:**



</details>

******