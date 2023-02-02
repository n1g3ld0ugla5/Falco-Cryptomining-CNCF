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

## Untaint the control plane
```
kubectl taint node ip-10-0-2-104 node-role.kubernetes.io/control-plane:NoSchedule-
```

## Moving the kubeconfig locally for port forwarding Falco Sidekick

Sanity checks
```
cd $HOME/.kube/
ls
kubectl config view
cat config
exit
kubeadm kubeconfig
cd $HOME/.kube/
echo $KUBECONFIG
exit
```

Use the exported kubeconfig file locally
Using the public IP of the EC2 instance.
```
vi config.yaml
export KUBECONFIG=./config.yaml
kubectl config get-contexts 
```

Access the files locally
```
kubectl get pods -A --insecure-skip-tls-verify
```

Remember to install the Falco Sidekick UI
```
helm upgrade falcosidekick falcosecurity/falcosidekick --set webui.enabled=true --kube-insecure-skip-tls-verify
```

WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: ./config.yaml
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: ./config.yaml <br/>
Release "falcosidekick" has been upgraded. Happy Helming! <br/>
NAME: falcosidekick <br/>
LAST DEPLOYED: Wed Feb  1 14:58:52 2023 <br/>
NAMESPACE: default <br/>
STATUS: deployed <br/>
REVISION: 2

## In case sidekick redis fails to install due to missing pvc
0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. <br/>
preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..

```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml --insecure-skip-tls-verify
```

namespace/local-path-storage created <br/>
serviceaccount/local-path-provisioner-service-account created <br/>
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created <br/>
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created <br/>
deployment.apps/local-path-provisioner created <br/>
storageclass.storage.k8s.io/local-path created <br/>
configmap/local-path-config created

```
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}' --insecure-skip-tls-verify
```
storageclass.storage.k8s.io/local-path patched

```
kubectl get storageclass --insecure-skip-tls-verify
```

NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  119s
standard               kubernetes.io/aws-ebs   Retain          Immediate              true                   21h

## Finally, Port-Foward the Falco Sidekick from Macbook
```
kubectl port-forward svc/falcosidekick-ui 2802 --insecure-skip-tls-verify
```

Forwarding from 127.0.0.1:2802 -> 2802
Forwarding from [::1]:2802 -> 2802
Handling connection for 2802
Handling connection for 2802
Handling connection for 2802
Handling connection for 2802
Handling connection for 2802
Handling connection for 2802


<img width="1160" alt="Screenshot 2023-02-01 at 15 39 35" src="https://user-images.githubusercontent.com/109959738/216095806-928191c1-d85d-43a8-95d8-186669abe0d2.png">

```
kubectl create namespace miner-test --insecure-skip-tls-verify
```
```
kubectl apply -f miner.yaml -n miner-test --insecure-skip-tls-verify
```

## Testing Falco Rules

Run the following command to trigger one of the Falco rules:
```
sudo find /root -name "id_rsa"
```

However, it did not trigger the event:
```
grep falco /var/log/syslog | grep "find /root -name id_rsa"
```

Removing the specific event grep, I get a lot of output data: <br/>
It's all regarding some misconfigurations I made on Sidekick earlier:
```
grep falco /var/log/syslog
```

Feb  2 11:22:35 ip-10-0-2-104 containerd[504]: 2023-02-02 11:22:35.741 [WARNING][5655] <br/>
ipam_plugin.go 433: Asked to release address but it doesn't exist. <br/>
Ignoring ContainerID="0f49213dc2015af9c6eda56cd5f5e3d421673b0b986549bed0dac58e87f482b6" <br/>
HandleID="k8s-pod-network.0f49213dc2015af9c6eda56cd5f5e3d421673b0b986549bed0dac58e87f482b6" <br/>
Workload="ip--10--0--2--104-k8s-falcosidekick--ui--5c566c599b--xl446-eth0" <br/>
<br/>
Feb  2 11:22:35 ip-10-0-2-104 containerd[504]: 2023-02-02 11:22:35.741 [INFO][5655] <br/>
ipam_plugin.go 444: Releasing address using workloadID <br/>
ContainerID="0f49213dc2015af9c6eda56cd5f5e3d421673b0b986549bed0dac58e87f482b6" <br/>
HandleID="k8s-pod-network.0f49213dc2015af9c6eda56cd5f5e3d421673b0b986549bed0dac58e87f482b6" <br/>
Workload="ip--10--0--2--104-k8s-falcosidekick--ui--5c566c599b--xl446-eth0"

## Investigating Further

Wait for the Falco pods to be in running state. <br/>
Then, check which source of events is configured.
```
kubectl logs -n falco -l app.kubernetes.io/name=falco -c falco-driver-loader --tail=-1 \
  | grep "* Running falco-driver-loader with"
```

```* Running falco-driver-loader with:``` driver=module, compile=yes, download=yes <br/>
The driver=module confirms that the Kernel Module is being used as the source for syscall events. <br/>
The command returns one log for each pod.

```
kubectl logs -n falco -l app.kubernetes.io/name=falco -c falco-driver-loader --tail=-1 \
  | grep "* Success"
```

```* Success:``` falco module found and inserted <br/>
<br/>

To do so, let's simulate someone trying to sniff for SSH keys. <br/>
Run find on the root home dir, querying for ```"id_rsa"```:

```
find /root -name "id_rsa"
```

This action triggers the rule ```Search Private Keys or Passwords```. <br/> 
Print to stdout the logs with:

```
kubectl logs -n falco -l app.kubernetes.io/name=falco \
  | grep "Warning Grep private keys"
```

Defaulted container "falco" out of: falco, falco-driver-loader (init)
