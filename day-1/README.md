# 🧭 Day 1 – EClickstream Analytics Platformu Kurulumu  
> **90 Günlük Data Engineering Path — Gün 1**

## 🚀 Proje Özeti  

Bu proje, **Flink**, **Kafka**, **Airflow**, **Redis**, **Docker** ve **Kubernetes** bileşenlerini bir araya getirerek gerçek-zamanlı ve batch veri işleme yeteneği sunan uçtan uca bir **veri platformu** tasarlar.  

Örnek senaryo:  
Kullanıcı davranışı ve satış işlemleri gerçek zamanlı olarak takip edilir — mağaza sitesi veya mobil uygulamadan gelen **tıklama**, **sepete ekleme**, **sipariş verme** ve **ödeme** olayları Kafka topic’lerine yazılır.  
**Flink**, bu event’leri event-time pencereleriyle işler; **sipariş sayısı**, **konversiyon oranı**, **sepet terk etme oranı**, **anlık ciro** ve **ürün bazlı satış** gibi KPI’lar üretir.  

**Airflow**, batch süreçleri (ör. stok raporları, fraud backfill, arşivleme) orkestre eder.  
Sonuçlar **Redis** üzerinde milisaniye düzeyinde sorgulanabilir hale gelir.  
**Docker** ve **Kubernetes**, tüm sistemi ölçeklenebilir ve fault-tolerant şekilde çalıştırır.  

🎯 Amaç: Firmalarının anlık içgörüler elde edip **dinamik fiyatlama**, **kişiselleştirilmiş öneriler** ve **fraud tespiti** yapabileceği bir modern veri platformu inşa etmek.  

---

## 🧩 Platformun Omurgası

| Katman | Teknoloji | Amaç |
|--------|------------|------|
| Stream Processing | Apache Flink | Gerçek zamanlı veri işleme |
| Message Queue | Apache Kafka | Event akışı ve decoupling |
| Workflow Orchestration | Apache Airflow | Batch job yönetimi |
| In-Memory Storage | Redis | Düşük gecikmeli KPI saklama |
| Containerization | Docker | Servislerin paketlenmesi |
| Orchestration | Kubernetes (OpenShift CRC) | Dağıtık yönetim ve scaling |

---

## 🏗️ Adım 1 – OpenShift Ortamı Hazırlığı  

Bu proje **CodeReady Containers (CRC)** üzerinde çalışıyor.  
CRC, geliştiriciler için tek düğümlü bir **OpenShift 4 kümesi** kurar — masaüstünde, üretim ortamına yakın deneyim sağlar.  

```bash
crc start
oc login -u kubeadmin https://api.crc.testing:6443
oc new-project ecom-data-platform
```

Artık tüm bileşenleri `ecom-data-platform` namespace’i altında yöneteceğiz.  

---

## 📨 Adım 2 – Strimzi Kafka Cluster Kurulumu  

Kafka’yı doğrudan yönetmek yerine **Strimzi Operator** kullanıyoruz.  
Strimzi, Kafka’yı Kubernetes üzerinde deklaratif olarak yönetmeyi sağlar.  
Elle StatefulSet kurulumuna göre çok daha pratik ve hatasızdır.  

### 📄 `kafka-cluster.yaml`
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kfk
  namespace: ecom-data-platform
spec:
  kafka:
    version: 4.1.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
    storage:
      type: ephemeral
  zookeeper:
    replicas: 1
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

Kurulum:  
```bash
oc apply -f https://strimzi.io/install/latest?namespace=ecom-data-platform
oc apply -f kafka-cluster.yaml
```

Producer / Consumer testleri:  
```bash
# Producer
oc -n ecom-data-platform run kafka-producer --rm -it --restart=Never   --image=quay.io/strimzi/kafka:0.48.0-kafka-4.1.0 --command --   sh -c 'echo "{"userId":123,"action":"view","product":"Shoes"}" | bin/kafka-console-producer.sh --bootstrap-server kfk-kafka-bootstrap:9092 --topic click-stream-ecom'

# Consumer
oc -n ecom-data-platform run kafka-consumer --rm -it --restart=Never   --image=quay.io/strimzi/kafka:0.48.0-kafka-4.1.0 --command --   sh -c 'bin/kafka-console-consumer.sh --bootstrap-server kfk-kafka-bootstrap:9092 --topic click-stream-ecom --from-beginning --max-messages 1'
```

✅ **Beklenen çıktı:**  
```json
{"userId":123,"action":"view","product":"Shoes"}
```

---

## ⚡ Adım 3 – Flink Cluster Kurulumu  

### 📄 `flink-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flink-conf
  namespace: ecom-data-platform
data:
  flink-conf.yaml: |
    jobmanager.rpc.address: flink-jobmanager
    rest.address: flink-jobmanager
    parallelism.default: 1
    taskmanager.numberOfTaskSlots: 1
    jobmanager.memory.process.size: 768m
    taskmanager.memory.process.size: 704m
```

### 📄 `flink-jobmanager.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-jobmanager
  namespace: ecom-data-platform
  labels:
    app: flink
    role: jobmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flink
      role: jobmanager
  template:
    metadata:
      labels:
        app: flink
        role: jobmanager
    spec:
      containers:
        - name: jm
          image: docker.io/library/flink:1.18.1-scala_2.12-java11
          ports:
            - containerPort: 6123
            - containerPort: 8081
          volumeMounts:
            - name: conf
              mountPath: /opt/flink/conf
      volumes:
        - name: conf
          configMap:
            name: flink-conf
---
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager
  namespace: ecom-data-platform
spec:
  ports:
    - port: 8081
      name: webui
  selector:
    app: flink
    role: jobmanager
```

### 📄 `flink-taskmanager.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-taskmanager
  namespace: ecom-data-platform
  labels:
    app: flink
    role: taskmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flink
      role: taskmanager
  template:
    metadata:
      labels:
        app: flink
        role: taskmanager
    spec:
      containers:
        - name: tm
          image: docker.io/library/flink:1.18.1-scala_2.12-java11
          ports:
            - containerPort: 6122
          volumeMounts:
            - name: conf
              mountPath: /opt/flink/conf
      volumes:
        - name: conf
          configMap:
            name: flink-conf
```

Flink UI erişimi:
```bash
oc expose svc/flink-jobmanager --port=8081 --target-port=8081 --name=flink-ui
oc get route flink-ui -o jsonpath='{.spec.host}{"\n"}'
```

---

## 🗄️ Adım 4 – Redis Kurulumu  

### 📄 `redis-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ecom-data-platform
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: docker.io/redis:7.2-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: ecom-data-platform
spec:
  ports:
    - port: 6379
  selector:
    app: redis
```

---

## 🕹️ Adım 5 – Airflow Hazırlığı  

### 📄 `airflow-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: airflow
  namespace: ecom-data-platform
spec:
  replicas: 1
  selector:
    matchLabels:
      app: airflow
  template:
    metadata:
      labels:
        app: airflow
    spec:
      containers:
        - name: webserver
          image: apache/airflow:2.9.0
          ports:
            - containerPort: 8080
          env:
            - name: AIRFLOW__CORE__LOAD_EXAMPLES
              value: "false"
            - name: AIRFLOW__CORE__EXECUTOR
              value: "SequentialExecutor"
          command: ["bash", "-c"]
          args: ["airflow db init && airflow users create --username admin --password admin --firstname Admin --lastname User --role Admin --email admin@example.com && airflow webserver"]
---
apiVersion: v1
kind: Service
metadata:
  name: airflow-webserver
  namespace: ecom-data-platform
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: airflow
```

Airflow UI erişimi:
```bash
oc expose svc/airflow-webserver --port=8080 --target-port=8080 --name=airflow-ui
oc get route airflow-ui -o jsonpath='{.spec.host}{"\n"}'
```

---

## 🔍 Mimarinin Görseli  

![Tam Entegre E-Commerce Platformu](images/Tam_Entegre.png)

> Yukarıdaki diyagram, e-commerce clickstream akışını Kafka → Flink → Redis hattında gösteriyor.  
> Airflow batch süreçlerini orkestre ederken Kubernetes tüm bileşenlerin dayanıklılığını sağlar.

---

## 📚 Devam  

Bu **1. Gün**: ortam kurulumu tamamlandı.  
**2. Gün**'de Kafka topic’lerinden Flink stream job’una gerçek-zamanlı veri akışını oluşturacağız.  
