# kubernetes-ELK

在k8s上部署ELK，其中包含metricbeat,filebeat以及eventrouter等相關元件

## 利用elk監控k8s叢集
1. 資源使用量(如:CPU、Memory等)
2. k8s log
3. k8s 


---


### 建立kube-state-metrics
透過kube-state-metrics將k8s資源使用量取出


clone專案:
```shell=
git clone https://github.com/wheam2002/kubernetes-ELK.git
```

至kube-state-metrics路徑中並使用yaml部署kube-state-metrics:
```shell=
cd ~/ç/kube-state-metrics/examples/standard/
kubectl apply -f .
```

### 建立elasticsearch

至kubernetes-ELK路徑中部署elasticsearch:
```shell=
cd ~/kubernetes-ELK
kubectl apply -f elasticsearch.yaml
```

### 建立kibana

至kubernetes-ELK路徑中部署kibana:
```shell=
cd ~/kubernetes-ELK
kubectl apply -f kibana.yaml
```

更改kibana的對外port

* kibana.yaml
```yaml=
...(略)
spec:
  type: NodePort 
  ports:
  - port: 5601
    nodePort: 32700 # nodePort(對外port)欄位可自行修改
    protocol: TCP
    targetPort: ui
  selector:
    k8s-app: kibana
...
```
### 建立metricbeat

metricbeat負責將k8s metric送至elasticsearch

至kubernetes-ELK路徑中部署metricbeat:
```shell=
cd ~/kubernetes-ELK
kubectl apply -f metricbeat.yaml
```

### 建立log-filebeat

log-filebeat負責將log送至elasticsearch

至kubernetes-ELK路徑中部署log-filebeat:
```shell=
cd ~/kubernetes-ELK
kubectl apply -f log-filebeat.yaml
```

### 建立eventrouter-filebeat

此pod中包含eventrouter和filebeat，其中eventrouter負責收集k8s的events，filebeat則負責傳送log至elasticsearch或是logstash (此處是傳往logstash)

修改eventrouter的Dockerfile，預設不會將events存成file，修改成將events存成file，此處的Dockerfile以修改成存成file的形式

eventrouter的Dockerfile如下(file形式)
```dockerfile=
FROM openshift/origin-release:golang-1.14 AS build
COPY . /go/src/github.com/openshift/eventrouter
RUN cd /go/src/github.com/openshift/eventrouter && go build .
FROM centos:7
RUN mkdir -p /data/log/eventrouter
COPY --from=build /go/src/github.com/openshift/eventrouter/eventrouter /bin/eventrouter
# 原本的指令
CMD ["/bin/eventrouter", "-v", "3", "-logtostderr"]
# 修改後
CMD ["/bin/eventrouter", "-v", "3", "-log_dir", "/data/log/eventrouter"]
```

至kubernetes-ELK路徑中部署eventrouter-filebeat:
```shell=
cd ~/kubernetes-ELK
kubectl apply -f eventrouter-filebeat.yaml
```

### 建立logstash

logstash負責接收eventrouter-filebeat所傳送的events，對events做資料整理過後再送往elasticsearch

至kubernetes-ELK路徑中部署logstash:
```shell=
cd ~/kubernetes-ELK
kubectl apply -f logstash.yaml
```

