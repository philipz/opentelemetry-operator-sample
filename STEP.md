[Use Public NAT with GKE](https://cloud.google.com/nat/docs/gke-example)
https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters
https://cloud.google.com/kubernetes-engine/docs/how-to/authorized-networks

1. `gcloud config set project cloudrun-s001`

2. `gcloud compute networks create custom-network1 --subnet-mode custom`

3. 
```
gcloud container clusters create-auto private-cluster-0 \
    --create-subnetwork name=subnet-custom-network1 \
    --enable-master-authorized-networks \
    --enable-private-nodes \
    --enable-private-endpoint \
    --master-ipv4-cidr 172.16.0.32/28 \
    --region=asia-east1 \
    --network custom-network1
```

5.
```
gcloud compute firewall-rules create allow-ssh \
    --network custom-network1 \
    --source-ranges 35.235.240.0/20 \
    --allow tcp:22
```

6.
```
gcloud compute routers create nat-router \
    --network custom-network1 \
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

gcloud container clusters update private-cluster-0 \
    --enable-master-authorized-networks \
    --master-authorized-networks cloudshell,35.201.155.184/32




3. 
```
gcloud compute networks subnets create subnet-asia-east-1 \
   --network custom-network1 \
   --region asia-east1 \
   --range 192.168.1.0/24
```