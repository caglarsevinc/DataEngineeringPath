# Infrastructure Deployment Guide

## Introduction
This guide covers the deployment of the following technologies on Kubernetes using CRC and Docker Compose:
- Apache Kafka
- Apache Flink
- Redis
- Apache Airflow

## Prerequisites
- Ensure you have a Kubernetes cluster running (CRC or other).
- Install Docker and Docker Compose.
- Have `kubectl` installed and configured to interact with your cluster.

## Apache Kafka Deployment
### Using Kubernetes
1. Create a Kafka namespace:
   ```bash
   kubectl create namespace kafka
   ```
2. Deploy Zookeeper:
   ```bash
   kubectl apply -f https://bit.ly/2W1n16a
   ```
3. Deploy Kafka:
   ```bash
   kubectl apply -f https://bit.ly/2W2h1tv
   ```
4. Verify deployment:
   ```bash
   kubectl get pods -n kafka
   ```

### Using Docker Compose
1. Create a `docker-compose.yml` file:
   ```yaml
   version: '2'
   services:
     zookeeper:
       image: wurstmeister/zookeeper:3.4.6
       ports:
         - "2181:2181"
     kafka:
       image: wurstmeister/kafka:latest
       ports:
         - "9092:9092"
       expose:
         - "9093"
       environment:
         KAFKA_ADVERTISED_LISTENERS: INSIDE://kafka:9093,OUTSIDE://kafka:9092
         KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
         KAFKA_LISTENERS: INSIDE://0.0.0.0:9093,OUTSIDE://0.0.0.0:9092
         KAFKA_ZOOKEEPER: zookeeper:2181
   ```
2. Run the deployment:
   ```bash
   docker-compose up -d
   ```

## Apache Flink Deployment
### Using Kubernetes
1. Deploy Flink with Helm:
   ```bash
   helm repo add flink https://apache.jfrog.io/artifactory/flink
   helm install flink-release flink/flink
   ```

### Using Docker Compose
1. Create a Docker Compose file:
   ```yaml
   version: '2'
   services:
     jobmanager:
       image: flink:latest
       ports:
         - "8081:8081"
       command: jobmanager
     taskmanager:
       image: flink:latest
       command: taskmanager
       depends_on:
         - jobmanager
   ```
2. Run the deployment:
   ```bash
   docker-compose up -d
   ```

## Redis Deployment
### Using Kubernetes
1. Deploy Redis:
   ```bash
   kubectl create deployment redis --image=redis
   ```
2. Expose Redis:
   ```bash
   kubectl expose deployment redis --type=LoadBalancer --port=6379
   ```

### Using Docker Compose
1. Create a Docker Compose file:
   ```yaml
   version: '2'
   services:
     redis:
       image: redis:latest
       ports:
         - "6379:6379"
   ```
2. Run the deployment:
   ```bash
   docker-compose up -d
   ```

## Apache Airflow Deployment
### Using Kubernetes
1. Deploy Airflow with Helm:
   ```bash
   helm repo add apache-airflow https://apache.github.io/airflow-site/helm-chart/stable
   helm install airflow apache-airflow/airflow
   ```

### Using Docker Compose
1. Create a Docker Compose file:
   ```yaml
   version: '2'
   services:
     airflow:
       image: apache/airflow:latest
       restart: always
       ports:
         - "8080:8080"
       depends_on:
         - redis
         - postgres
     redis:
       image: redis:latest
     postgres:
       image: postgres:latest
   ```
2. Run the deployment:
   ```bash
   docker-compose up -d
   ```

## Conclusion
This guide provides you with a starting point to deploy Kafka, Flink, Redis, and Airflow on Kubernetes and Docker Compose. Adjust configurations according to your project needs.