# OpenTelemetry on GCP GKE安裝步驟

## 參考資料

https://cloud.google.com/blog/topics/developers-practitioners/easy-telemetry-instrumentation-gke-opentelemetry-operator/
https://cloud.google.com/kubernetes-engine/docs/tutorials/private-cluster-bastion
https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters
https://github.com/GoogleCloudPlatform/opentelemetry-operator-sample

## 建立實驗專案環境

### 建立GKE Private Cluster，若已有Cluster可跳過

1. 若專案為*cloudrun-s001*，設定為預設專案

    `gcloud config set project cloudrun-s001`

2. 建立VPC *custom-network*供Private Cluster使用
    `gcloud compute networks create custom-network --subnet-mode custom`

3. 建立Private Cluster
    A. 使用enable-private-nodes和enable-private-endpoint
    ```
    gcloud container clusters create-auto private-cluster \
        --create-subnetwork=name=subnet-custom-network \
        --enable-master-authorized-networks \
        --enable-private-nodes \
        --enable-private-endpoint \
        --master-ipv4-cidr 172.16.0.0/28 \
        --region=asia-east1 \
        --network custom-network
    ```
    
    B. 只使用enable-private-nodes，不使用enable-private-endpoint
    ```
    gcloud container clusters create-auto private-cluster \
        --create-subnetwork=name=subnet-custom-network \
        --enable-master-authorized-networks \
        --enable-private-nodes \
        --master-ipv4-cidr 172.16.0.0/28 \
        --region=asia-east1 \
        --network custom-network
    ```

    如果只使用enable-private-nodes，不使用enable-private-endpoint，則須設定可以存取control plane的網段
    加上A3的兩組連GCP IP: 123.51.165.160 & 61.216.71.43
    ```
    gcloud container clusters update private-cluster \
        --enable-master-authorized-networks \
        --master-authorized-networks 123.51.165.160/32
    ```

    ```
    gcloud container clusters update private-cluster \
        --enable-master-authorized-networks \
        --master-authorized-networks 61.216.71.43/32
    ```
    不使用enable-private-endpoint，則Step 6~9可跳過，直接到進行安裝OpeneTelemetry Operator

4. 建立VPC內的Cloud Router提供連線到外部服務
    ```
    gcloud compute routers create nat-router \
        --network custom-network \
        --region asia-east1
    ```

5. 新增路由器配置允許自動為NAT閘道指派外部IP位址
    ```
    gcloud compute routers nats create nat-config \
        --router-region asia-east1 \
        --router nat-router \
        --nat-all-subnet-ip-ranges \
        --auto-allocate-nat-external-ips
    ```

6. 設定防火牆允許 Identity-Aware Proxy 網段連入
    ```
    gcloud compute firewall-rules create allow-ssh \
        --network custom-network \
        --source-ranges 35.235.240.0/20 \
        --allow tcp:22
    ```

7. 建立跳板機(GKE使用enable-private-endpoint才需要)
    ```
    gcloud compute instances create bastion-vm \
        --zone=asia-east1-c \
        --machine-type=e2-micro \
        --network-interface=no-address,network-tier=PREMIUM,subnet=subnet-custom-network
    ```

8. 在跳板機安裝代理程式Tinyproxy

    建立跳板機和GKE Private Cluster後，必須在跳板機中部署Proxy代理程式Tinyproxy，以將流量轉送至叢集控制層。
    1. SSH登入到跳板機內：
    ```
    gcloud compute ssh bastion-vm --tunnel-through-iap --project=cloudrun-s001
    ```
    2. 安裝Tinyproxy：
    `sudo apt install tinyproxy`
    3. 設定Tinyproxy配置檔案：
    `sudo vi /etc/tinyproxy/tinyproxy.conf`
    4. 在配置檔案中，執行以下作業：
    驗證Port是否為8888。
    5. VIM搜尋 Allow 部分：
    `/Allow 127`
    6. 將下一行設定到 Allow 段落：
    `Allow localhost`
    7. 儲存配置檔並重新啟動Tinyproxy：
    `sudo service tinyproxy restart`
    8. 登出跳板機：
    `exit`

9. 從遠端Client連線到叢集

    完成 Tinyproxy 配置後，在Client端先取得叢集存取憑證並指定代理位址，在遠端Client上執行以下操作：
    1. 獲取叢集存取憑證：
    ```
    gcloud container clusters get-credentials private-cluster \
        --region=asia-east1 \
        --project=cloudrun-s001
    ```
    2. 使用 IAP 透過Tunnel連線到跳板機：
    ```
    gcloud compute ssh bastion-vm \
        --tunnel-through-iap \
        --project=cloudrun-s001 \
        --zone=asia-east1-c \
        --ssh-flag="-4 -L8888:localhost:8888 -N -q -f"
    ```
    3. 指定代理位址：
    `export HTTPS_PROXY=localhost:8888`
    4. 測試kubectl輸出目前叢集的namespace：
    `kubectl get ns`

## 安裝OpeneTelemetry Operator

1. 查詢目前GKE防火牆規則，將private-cluster替換成正確叢集名稱
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

2. 開通cert-manager firewall-rule，將172.16.0.0/28和gke-private-cluster-0ed1d9bf-node改為上面查詢的結果
    ```
    gcloud compute firewall-rules create cert-manager-9443 \
    --source-ranges 172.16.0.0/28 \
    --target-tags gke-private-cluster-0ed1d9bf-node \
    --allow TCP:9443
    ```

3. 安裝 cert-manager，必須有[Helm](https://helm.sh/)工具
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

4. 安裝 OpenTelemetry Operator
    `kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml`

## 建立 Workload Identity 以供IAM service account對應到GKE service account

出處：https://github.com/GoogleCloudPlatform/opentelemetry-operator-sample/tree/main/recipes/cloud-trace

1. 建立otel-collector service account
    ```
    export GCLOUD_PROJECT=cloudrun-s001
    gcloud iam service-accounts create otel-collector --project=${GCLOUD_PROJECT}
    ```

2. 綁定Cloud Trace Agent角色
    ```
    gcloud projects add-iam-policy-binding $GCLOUD_PROJECT \
        --member "serviceAccount:otel-collector@${GCLOUD_PROJECT}.iam.gserviceaccount.com" \
        --role "roles/cloudtrace.agent"
    ```

3. 綁定Cloud Monitoring Metric Writer角色
    ```
    gcloud projects add-iam-policy-binding $GCLOUD_PROJECT \
        --member "serviceAccount:otel-collector@${GCLOUD_PROJECT}.iam.gserviceaccount.com" \
        --role "roles/monitoring.metricWriter"
    ```
4. 綁定Cloud logging log Writer角色
    ```
    gcloud projects add-iam-policy-binding $GCLOUD_PROJECT \
        --member "serviceAccount:otel-collector@${GCLOUD_PROJECT}.iam.gserviceaccount.com" \
        --role "roles/logging.logWriter"
    ```
5. 設定COLLECTOR收集的namespace，並綁定到GKE default namespece的otel-collector service account，請將default改成自己的namespace
    ```
    export COLLECTOR_NAMESPACE=default
    gcloud iam service-accounts add-iam-policy-binding "otel-collector@${GCLOUD_PROJECT}.iam.gserviceaccount.com" \
        --role roles/iam.workloadIdentityUser \
        --member "serviceAccount:${GCLOUD_PROJECT}.svc.id.goog[${COLLECTOR_NAMESPACE}/otel-collector]"
    ```

## 建立並設定 OpenTelemetry Collector

1. 新增 otel-collector 的部署檔 collector-config-new.yaml
```
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel
spec:
  image: otel/opentelemetry-collector-contrib:latest
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 65
        spike_limit_percentage: 20

    exporters:
      googlecloud:
      logging:
        loglevel: debug

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [logging, googlecloud]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [googlecloud]
        logs:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [googlecloud]
```

2. 建立 OpenTelemetry Collector
    `kubectl apply -f collector-config-new.yaml`

3. 添加 annotation 到 OpenTelemetry Collector
    ```
    kubectl annotate opentelemetrycollectors otel \
        --namespace $COLLECTOR_NAMESPACE \
        iam.gke.io/gcp-service-account=otel-collector@${GCLOUD_PROJECT}.iam.gserviceaccount.com
    ```

4. 建立 Instrumentation
    `kubectl apply -f instrumentation.yaml`

## 應用程式YAML加上auto instrumentation的annotation

在同一個namespace下的deployment/statefulset的YAML檔下的spec.template.metadata.annotations加上*instrumentation.opentelemetry.io/inject-java: "true"*
```
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
```

如此就可以在Cloud Trace、Cloud Logging和Monitoring Metrics查到相關可觀測的數據。