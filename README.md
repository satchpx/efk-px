# efk-px
EFK stack backed by Portworx on Kubernetes

## Install Portworx
```
https://docs.portworx.com/portworx-install-with-kubernetes/
```

## Install Elasticsearch
### Install helm
```
https://github.com/helm/helm/blob/master/docs/install.md
```

### Initialize helm
```
helm  init
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

### Create the required storageClasses
```
kubectl apply -f manifests/portworx-storageclasses.yaml
```

### Deploy elasticsearch helm chart
```
helm install --name elasticsearch-master --values manifests/es-master-values-px-rf3.yaml ../helm-charts/elasticsearch
helm install --name elasticsearch-data --values manifests/es-client-values-px-rf3.yaml ../helm-charts/elasticsearch
helm install --name kibana --values manifests/kibana-values.yaml ../helm-charts/kibana
```

### Create a service for kibana to expose it outside the kubernetes cluster
* NOTE: this is only for on-prem, skip this for cloud
```
apiVersion: v1
kind: Service
metadata:
  name: kibana-np
  labels:
    app: kibana
spec:
  type: NodePort
  ports:
    - port: 5601
  selector:
    app: kibana
```
### Install Fluent-Bit
```
helm install --name fluent-bit --values manifests/fluent-bit-values.yaml helm-charts/fluent-bit
```

### Install Elasticsearch Exporter
```
helm install --name elasticsearch-exporter ../helm-charts/elasticsearch-exporter --set es.uri=http://elasticsearch-master.default:9200
```

### Update the exporter service to have it scraped by prometheus
```
kubectl edit svc elasticsearch-exporter
```
....and add the following annotation under metadata:
```
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
```


### Install prometheus and grafana for monitoring
```
git clone https://github.com/satchpx/prom-helm.git
cd prom-helm/
helm install --name prometheus --namespace monitoring prometheus --set server.persistentVolume.storageClass=px-db-rf1-sc,server.shedulerName=stork
helm install --name grafana --namespace monitoring grafana --set persistence.enabled=true,persistence.storageClassName=px-db-rf1-sc,schedulerName=stork
```

### Expose loadbalancer service for kibana and grafana
```
kubectl apply -f manifests/kibana-lb.yaml
kubectl apply -f manifests/grafana-lb.yaml -n monitoring
```

