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

helm install falco falcosecurity/falco --namespace falco --create-namespace
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

## Install the Cryptominer

