# Elastic / Rancher Workshop

Rancher is a fantastic Kubernetes distribution, particularly for air gapped environments!
Rancher has a [workshop](https://github.com/clemenko/rke_workshop) that they typically use to show how to get going with three virtual machines and be fully operating with a STIG compliante Kubernetes environment.
This workshop is a companion to that workshop to show how to get Elastic Cloud for Kuberentes running in a Kubernetes environment and then deploy clusters on top of that environment.

The workshop will work through:
* [Environment Prep - Rancher workshop specific updates](#Environment-prep-work)
* [Lab 1: Install and First Cluster](#lab-1---eck-install--first-cluster)
* [Lab 2: Cross Cluster Search](#lab-2---cross-cluster-search)
* [Lab 3: Stack Monitoring & Upgrades](#lab-3---stack-monitoring--upgrades)

## Environment prep work

Let's create a directory to put files that we will need during the lab.

```bash
mkdir /opt/elastic-artifacts
cd /opt/elastic-artifacts
```

## Lab 1 - ECK Install & First Cluster

Elastic highly recommends managing Elastic Stack depoyments by using the Elastic Cloud for Kubernetes (ECK) operator.
Thus, let's start by installing the operator.

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

The operator is able to run with a basic (free) license or an enterprise license.
For the workshop, we will install a trial license, which covers thirty (30) days of use.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: eck-trial-license
  namespace: elastic-system
  labels:
    license.k8s.elastic.co/type: enterprise_trial
  annotations:
    elastic.co/eula: accepted 
EOF
```

### Create a tiny cluster

The following command will create a starter Elastic Stack deployment with a single Elasticsearch node and a single Kibana instance.
The Kibana service will be TLS secured and made available on all of the Kubernets hosts.

```bash
helm install es-kb-quickstart elastic/eck-stack -n elastic-stack --create-namespace --set eck-kibana.spec.http.service.spec.type=NodePort
```

Wait for the cluster to become available: `kubectl -n elastic-stack get po`

### Log Into Kibana

Once the pods are ready, we will need two pieces of information from the deployment to access Kibana: the default Elastic user and the Kibana service port.

First get the `elastic` user password.

```bash
kubectl get secret -n elastic-stack elasticsearch-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' ; echo
```

Now get the port where Kibana is being served up:

```bash
kubectl get svc -n elastic-stack es-kb-quickstart-eck-kibana-kb-http
```

The above will emit an answer like the following.
In this example, Kibana is being served up on port 32117 on all hosts.

```bash
NAME                                  TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
es-kb-quickstart-eck-kibana-kb-http   NodePort   10.43.51.207   <none>        5601:32117/TCP   30m
```

We can further simplify this process by having the shell just produce the URL.

```bash
echo "https://$(hostname).rfed.run:$(kubectl get svc -n elastic-stack es-kb-quickstart-eck-kibana-kb-http -o jsonpath='{.spec.ports[].nodePort}')"
```

Open Kibana on one of the hosts, e.g. https://student$NUMa.rfed.run:$PORT.
Log into Kibana using `elastic` as the user name and the password that was generated during the installation.

Spend a couple of minutes exploring Kibana, e.g. [loading a sample data](assets/LoadingKibanaSampleData.pdf) set.

### Explore the Custom Resource Definition Outputs

With the creation of the custom resource definitions (CRDs), kubernetes has the ability to track the status of the top level concept of an Elasticsearch Cluster instead of just Pods.
Let's run the commands to get the status of the Elasticsearch and Kibana resources.
We will use the short form, e.g. "es" and "kb", for the API resource type.

Get the status of all of the Elasticsearch resources.

```bash
kubectl get es -A
```

Get the status of all of the Kibana resources.

```bash
kubectl get kb -A
```

### Outcome

With a few commands, we were able to deploy an SRE (Site Reliability Engineer) in a box to monitor our Elastic Stack.
In another command, we were able to deploy Elasticsearch and Kibana in a simple, development configuration.
The full Elastic Stack was secured from the start.
Finally, we were able to run some status commands to get insight into the cluster health right from the kubernetes command line.

### Clean Up

Go ahead and remove the initial cluster that was created. It will not be used in any of the subsequent labs.

```bash
helm uninstall es-kb-quickstart -n elastic-stack
```

## Lab 2 - Cross Cluster Search

In this lab, we will use the full YAML specification for the cluster definition.
In this example we will create two clusters and configure them to work together.

### Data Lake Cluster
The first cluster will represent a data lake in the environment. 
This cluster represents an environment where someone is authorized only to run searches.
First, create a file that defines the _data lake_ cluster.

```bash
cat <<EOF > datalake.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: data-lake

---

apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: data-lake
  namespace: data-lake
spec:
  nodeSets:
  - count: 3
    name: default
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 2Gi
              cpu: 0.5
            limits:
              memory: 2Gi
  version: 8.5.3

---

apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: data-lake
  namespace: data-lake
spec:
  version: 8.5.3
  http:
    service:
      spec:
        type: NodePort
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: 0.5
          limits:
            memory: 1Gi
            cpu: 0.5
  count: 1
  elasticsearchRef:
    name: data-lake
EOF
```

With the file in place, create the cluster:

```bash
kubectl apply -f datalake.yaml
```

We can track the deployment by observing the pods running in the defined namespace:

```bash
watch "kubectl get po -n data-lake ; kubectl get es -n data-lake"
```

#### Load Data into the Lake using Kibana

Once the pods are ready, we will need two pieces of information from the deployment to access Kibana: the default Elastic user and the Kibana service port.

First get the `elastic` user password.

```bash
kubectl get secret -n data-lake data-lake-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' ; echo
```

Now get the URL where Kibana is being served up:

```bash
echo "https://$(hostname).rfed.run:$(kubectl get svc -n data-lake data-lake-kb-http -o jsonpath='{.spec.ports[].nodePort}')"
```

Open Kibana on one of the hosts, e.g. https://student$NUMa.rfed.run:$PORT.
Log into Kibana using `elastic` as the user name and the password that was generated during the installation.

[Load the Kibana ecommerce sample data](assets/LoadingKibanaSampleData.pdf).

### Experimental Cluster

Let's create a second cluster that connects to the data lake to get data.
We will again start with creating Elastic Stack YAML definition.
Notice the change in the Elasticsearch definition.
The Elasticsearch definition now includes a specification to connect to the `data-lake` cluster.

```bash
cat <<EOF > experimental.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: experimental

---

apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: experimental
  namespace: experimental
spec:
  remoteClusters:
  - name: data-lake
    elasticsearchRef:
      name: data-lake
      namespace: data-lake
  nodeSets:
  - count: 3
    name: default
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 2Gi
              cpu: 0.5
            limits:
              memory: 2Gi
  version: 8.5.3

---

apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: experimental
  namespace: experimental
spec:
  version: 8.5.3
  http:
    service:
      spec:
        type: NodePort
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: 0.5
          limits:
            memory: 1Gi
            cpu: 0.5
  count: 1
  elasticsearchRef:
    name: experimental
EOF
```

With the file in place, create the cluster:

```bash
kubectl apply -f experimental.yaml
```

We can track the deployment by observing the pods running in the defined namespace:

```bash
watch "kubectl get po -n experimental ; kubectl get es -n experimental"
```

#### Search the data lake from the experimental cluster

Once the pods are ready, we will need two pieces of information from the deployment to access Kibana: the default Elastic user and the Kibana service port.

First get the `elastic` user password.

```bash
kubectl get secret -n experimental experimental-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' ; echo
```

Now get the URL where Kibana is being served up:

```bash
echo "https://$(hostname).rfed.run:$(kubectl get svc -n experimental experimental-kb-http -o jsonpath='{.spec.ports[].nodePort}')"
```


Open Kibana on one of the hosts, e.g. https://student$NUMa.rfed.run:$PORT.
Log into Kibana using `elastic` as the user name and the password that was generated during the installation.

Navigate to Dev Tools.

From within Dev Tools, run the following query:

```elasticsearch
GET data-lake:kibana_sample_data_ecommerce/_search
```

### Extra Credit

Open Rancher and check the CPU and RAM usage for each node.
Can we provision another cluster on these nodes?

### Outcome

We were able to see another way of defining the Elastic Stack deployment using the CRD YAML instead of the Helm charts.
By simply declaring a small block of YAML config, we were able to get full trust between two clusters, which means that it is trivial to spin up a separate cluster for each workload and link related workloads together.

### Cleanup

Let's go ahead and remvoe the the two clusters involved in this lab.

```bash
kubectl delete -f datalake.yaml
kubectl delete -f experimental.yaml
```

## Lab 3 - Stack Monitoring & Upgrades

In this lab we will walk through a couple of concepts: stack monitoring and stack upgrades.
First we will create an _observability_ cluster that will receive logs and metrics about an Elasticsearch cluster.
Then we will create a _mission_ cluster to that will be monitored by the observability cluster.
Lastly, we will upgrade the _mission_ cluster to a new version and watch the cluster proceed through the upgrade.


### Create the Observability Cluster

Creating a cluster hopefully is becoming a comfortable process.
We will start with creating a simple deployment to receive signals about the status of a cluster.
We will again start with creating the file for the monitoring cluster.


```bash
cat <<EOF > monitoring.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

---

apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: monitoring
  namespace: monitoring
spec:
  nodeSets:
  - count: 3
    name: default
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 2Gi
              cpu: 0.5
            limits:
              memory: 2Gi
  version: 8.6.1

---

apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: monitoring
  namespace: monitoring
spec:
  version: 8.6.1
  http:
    service:
      spec:
        type: NodePort
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: 0.5
          limits:
            memory: 1Gi
            cpu: 0.5
  count: 1
  elasticsearchRef:
    name: monitoring
EOF
```

With the file in place, create the cluster:

```bash
kubectl apply -f monitoring.yaml
```

We can track the deployment by observing the pods running in the defined namespace:

```bash
watch "kubectl get po -n monitoring; kubectl get es -n monitoring"
```

### Create the Mission Cluster

The mission cluster is the one that is necessary for critical operations.
Thus, we need to monitor the mission cluster to ensure that we know the health of the cluster.
Let's create a mission cluster that sends monitoring data to the monitoring cluster.


```bash
cat <<EOF > mission.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mission

---

apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: mission
  namespace: mission
spec:
  nodeSets:
  - count: 3
    name: default
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 2Gi
              cpu: 0.5
            limits:
              memory: 2Gi
  version: 8.5.3
  monitoring:
    metrics:
      elasticsearchRefs:
      - name: monitoring
        namespace: monitoring 
    logs:
      elasticsearchRefs:
      - name: monitoring
        namespace: monitoring

---

apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: mission
  namespace: mission
spec:
  version: 8.5.3

  http:
    service:
      spec:
        type: NodePort
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: 0.5
          limits:
            memory: 1Gi
            cpu: 0.5
  count: 1
  elasticsearchRef:
    name: mission
  monitoring:
    metrics:
      elasticsearchRefs:
      - name: monitoring
        namespace: monitoring 
    logs:
      elasticsearchRefs:
      - name: monitoring
        namespace: monitoring
EOF
```

With the file in place, create the cluster:

```bash
kubectl apply -f mission.yaml
```

We can track the deployment by observing the pods running in the defined namespace:

```bash
watch "kubectl get po -n mission; kubectl get es -n mission"
```

### Check out the Monitoring Data

With the mission cluster deployed, open the Kibana on the monitoring cluster.
Gather the data for the `elastic` password and kibana URL.


```bash
kubectl get secret -n monitoring monitoring-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' ; echo
echo "https://$(hostname).rfed.run:$(kubectl get svc -n monitoring monitoring-kb-http -o jsonpath='{.spec.ports[].nodePort}')"
```

Once logged into Kibana, proceed to Stack Monitoring.
Stack Monitoring is accessible either using the top search bar inside Kibana or in the side panel as the next to last option.

Check on the health of the `mission` cluster.
What is the Elastic Stack version for the mission cluster?

### Load data into the Cluster

Open the mission cluster and [load the Kibana sample data](assets/LoadingKibanaSampleData.pdf).

```bash
kubectl get secret -n mission mission-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' ; echo
echo "https://$(hostname).rfed.run:$(kubectl get svc -n mission mission-kb-http -o jsonpath='{.spec.ports[].nodePort}')"
```

We will use that data to confirm that the data was retained through the upgrade that we are about to perform.

### Upgrade the mission Cluster

Upgrades for the Elastic Stack are well defined, and people have been successfully completing it by hand or with external DevOps practices.
Since the process is well defined, this process has been encoded into ECK.
Let's upgrade the mission cluster by merely changing the versions in the `mission.yaml` file.
The following command will update the version information for the Mission cluster.

```bash
sed -i 's/8.5.3/8.6.1/' mission.yaml
```

We can confirm the changne in VS Code or by running `cat mission.yaml`
Once we are happy with the change, apply it!

```bash
kubectl apply -f mission.yaml
```

While this upgrade is being applied, run the following command to see the changes that ECK is making to the Elastic Stack services.
Interacting with the data by opening one of the sample dashboards is a good way to see that the cluster remains available during the upgrade.

```bash
watch "kubectl get po -n mission; kubectl get es -n mission"
```

### Outcome

### Cleanup

Let's remove the resources that were built during this lab.

```bash
kubectl delete -f mission.yaml 
kubectl delete -f monitoring.yaml
```
