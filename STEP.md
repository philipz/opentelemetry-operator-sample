# OpenTelemetry on GCP GKE安裝步驟

## 參考資料

https://cloud.google.com/kubernetes-engine/docs/tutorials/private-cluster-bastion
https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters
https://github.com/GoogleCloudPlatform/opentelemetry-operator-sample

1. `gcloud config set project cloudrun-s001`

2. `gcloud compute networks create custom-network --subnet-mode custom`

3. 
```
gcloud container clusters create-auto private-cluster \
    --create-subnetwork=name=subnet-custom-network \
    --enable-master-authorized-networks \
    --enable-private-nodes \
    --enable-private-endpoint \
    --master-ipv4-cidr 172.16.0.32/28 \
    --region=asia-east1 \
    --network custom-network
```

5.
```
gcloud compute firewall-rules create allow-ssh \
    --network custom-network \
    --source-ranges 35.235.240.0/20 \
    --allow tcp:22
```

6.
```
gcloud compute routers create nat-router \
    --network custom-network \
    --region asia-east1
```

7.
```
gcloud compute routers nats create nat-config \
    --router-region asia-east1 \
    --router nat-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

8.
```
gcloud compute instances create bastion-vm \
    --zone=asia-east1-c \
    --machine-type=e2-micro \
    --network-interface=no-address,network-tier=PREMIUM,subnet=subnet-custom-network
```

9.
配置堡垒主机和专用集群后，您必须在主机中部署代理守护进程，以将流量转发到集群控制层面。在本教程中，您需要安装 Tinyproxy。
启动到虚拟机的会话：
```
gcloud compute ssh bastion-vm --tunnel-through-iap --project=cloudrun-s001
```
安装 Tinyproxy：
`sudo apt install tinyproxy`
打开 Tinyproxy 配置文件：
`sudo vi /etc/tinyproxy/tinyproxy.conf`
在该文件中，执行以下操作：
验证端口是否为 8888。
搜索 Allow 部分：
  /Allow 127
将以下行添加到 Allow 部分：
  Allow localhost
保存文件并重启 Tinyproxy：
`sudo service tinyproxy restart`
退出会话：
`exit`

10. 从远程客户端连接到集群
配置 Tinyproxy 后，您必须使用集群凭据设置远程客户端并指定代理。在远程客户端上执行以下操作：
获取集群的凭据：
```
gcloud container clusters get-credentials private-cluster \
    --region=asia-east1 \
    --project=cloudrun-s001
```
使用 IAP 通过隧道连接到堡垒主机：
```
gcloud compute ssh bastion-vm \
    --tunnel-through-iap \
    --project=cloudrun-s001 \
    --zone=asia-east1-c \
    --ssh-flag="-4 -L8888:localhost:8888 -N -q -f"
```
指定代理：
`export HTTPS_PROXY=localhost:8888`
输出是专用集群中的命名空间列表。
`kubectl get ns`

11. cert-manager firewall-rule
```
gcloud compute firewall-rules list \
    --filter 'name~^gke-private-cluster' \
    --format 'table(
        name,
        network,
        direction,
        sourceRanges.list():label=SRC_RANGES,
        allowed[].map().firewall_rule().list():label=ALLOW,
        targetTags.list():label=TARGET_TAGS
    )'
```

```
gcloud compute firewall-rules create cert-manager-9443 \
  --source-ranges 172.23.0.0/28 \
  --target-tags gke-gaas-lab-cluster-e520b76a-node \
  --allow TCP:9443
```

12. Install cert-manager

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  --create-namespace \
  --namespace cert-manager \
  --set installCRDs=true \
  --set global.leaderElection.namespace=cert-manager \
  --set extraArgs={--issuer-ambient-credentials=true} \
  cert-manager jetstack/cert-manager
```

13. Install OpenTelemetry Operator
`kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml`

14. Workload Identity
https://github.com/GoogleCloudPlatform/opentelemetry-operator-sample/tree/main/recipes/cloud-trace

```
export GCLOUD_PROJECT=cloudrun-s001
gcloud iam service-accounts create otel-collector --project=${GCLOUD_PROJECT}
```

```
gcloud projects add-iam-policy-binding $GCLOUD_PROJECT \
    --member "serviceAccount:otel-collector@${GCLOUD_PROJECT}.iam.gserviceaccount.com" \
    --role "roles/cloudtrace.agent; roles/monitoring.metricWriter; roles/logging.logWriter"
```

```
export COLLECTOR_NAMESPACE=default
gcloud iam service-accounts add-iam-policy-binding "otel-collector@${GCLOUD_PROJECT}.iam.gserviceaccount.com" \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:${GCLOUD_PROJECT}.svc.id.goog[${COLLECTOR_NAMESPACE}/otel-collector]"
```

15. Create An OpenTelemetry Collector
`kubectl apply -f new-otel.yaml`

16. Create An Instrumentation
`kubectl apply -f instrumentation.yaml`


17. Add annotation in deployment
  annotations:
    instrumentation.opentelemetry.io/inject-java: "true"