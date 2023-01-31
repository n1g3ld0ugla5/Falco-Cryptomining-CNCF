# Falco Cryptomining Lab - [CNCF]
This session will explain some of the intricacies of crypto-mining in Kubernetes and how crypto-jackers compromise containerized workloads at runtime. Cryptojacking is not limited to containers and hosts. <br/>

## Build a one-node Kubernetes Cluster with Falco on it
Run the below ``` bash ``` script to create a minimal ``` one ``` node ``` EKS ``` cluster

```
#!/bin/bash
export K8S_CLUSTER_NAME="nigel-mining-cluster"
export CLUSTER_REGION="eu-west-1"
export NODEGROUP_NAME="${K8S_CLUSTER_NAME}-nodegroup"
export INSTANCETYPE="m4.xlarge"
export NODES="1"

eksctl create cluster -n ${K8S_CLUSTER_NAME} \
  --nodegroup-name ${NODEGROUP_NAME} \
  --region ${CLUSTER_REGION} \
  --node-type ${INSTANCETYPE} \
  --nodes ${NODES}

helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco --namespace falco \
  --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true \
  --set auditLog.enabled=true
  
kubectl get pods -n falco -o wide -w
```

Get the logs from the pod ``` falco-rq4gs ```
```
kubectl logs falco-rq4gs -n falco
```

Mon Jan 30 10:56:26 2023: Falco version: 0.33.1 (x86_64) <br/>
Mon Jan 30 10:56:26 2023: Falco initialized with configuration file: /etc/falco/falco.yaml <br/>
Mon Jan 30 10:56:26 2023: Loading rules from file /etc/falco/falco_rules.yaml <br/>
Mon Jan 30 10:56:26 2023: Loading rules from file /etc/falco/falco_rules.local.yaml <br/>
Mon Jan 30 10:56:27 2023: The chosen syscall buffer dimension is: 8388608 bytes (8 MBs) <br/>
Mon Jan 30 10:56:27 2023: Starting health webserver with threadiness 4, listening on port 8765 <br/>
Mon Jan 30 10:56:27 2023: Enabled event sources: syscall <br/>
Mon Jan 30 10:56:27 2023: Opening capture with Kernel module <br/>

Check that Falco sidekick is working as expected
```
kubectl logs -n falco         falco-falcosidekick-ui-5b56bbd7cb-mtjkh
```

2023/01/31 15:25:00 [INFO] : Falcosidekick UI is listening on 0.0.0.0:2802
2023/01/31 15:25:00 [INFO] : log level is info

## Install the Cryptominer

Create a namespace for the miner
```
kubectl create namespace miner-test
```

Create a deployment called ``` deploy-miner.yaml``` with the below context to create the miner
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: miner
  name: miner
  namespace: miner-test
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: miner
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: miner
    spec:
      containers:
      - env:
        - name: POOL_URL
          value: pool.minexmr.com
        image: metal3d/xmrig:latest
        imagePullPolicy: Always
        name: xmrig
        resources:
          limits:
            cpu: 0.5
            memory: 4Gi
          requests:
            cpu: 0.25
            memory: 2Gi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```     
 
Apply the changes
```
kubectl apply -f deploy-miner.yaml -n miner-test
```


## Introduce the custom rules

So next step is to use the custom-rules.yaml file for installing the Falco Helm chart.
```
helm install falco -f custom-rules.yaml falcosecurity/falco
```

And we will see in our logs something like:
```
Mon Jan 30 10:56:26 2023: Loading rules from file /etc/falco/rules.d/rules-mining.yaml:
```


### Install on an EC2 instance
```
curl https://raw.githubusercontent.com/xxradar/install_k8s_ubuntu/main/setup.sh | bash          #K8SVERSION=1.25.5-00
curl https://raw.githubusercontent.com/xxradar/install_k8s_ubuntu/main/setup_latest.sh | bash
```

### Install Calico or Cilium as the Network Plugin
```
curl https://raw.githubusercontent.com/xxradar/install_k8s_ubuntu/main/calico_install.sh | bash
curl https://raw.githubusercontent.com/xxradar/install_k8s_ubuntu/main/cilium_install.sh | bash
```

### Install Helm
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Install Falco with Helm
```
helm install falco falcosecurity/falco --namespace falco   --create-namespace
```
