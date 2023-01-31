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

Create a deployment for the miner
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
kubectl apply -f miner-deployment.yaml
```


## Introduce the custom rules

We are going to create a file that contains custom rules so that we can keep it in a Git repository.
```
vi custom-rules.yaml
```

Remember to run the below command in ```vim``` before pasting the contents:
```
:set paste
```

Add the following content
```
customRules:
  rules-mining.yaml: |-

    # Threat Feed Objects

    - list: miner_ports
      items: [
         25, 3333, 3334, 3335, 3336, 3357, 4444,
         5555, 5556, 5588, 5730, 6099, 6666, 7777,
         7778, 8000, 8001, 8008, 8080, 8118, 8333,
         8888, 8899, 9332, 9999, 14433, 14444,
         45560, 45700
      ]

     - list: miner_domains
       items: [
         "asia1.ethpool.org","ca.minexmr.com",
         "cn.stratum.slushpool.com","de.minexmr.com",
         "eth-ar.dwarfpool.com","eth-asia.dwarfpool.com",
         "eth-asia1.nanopool.org","eth-au.dwarfpool.com",
         "eth-au1.nanopool.org","eth-br.dwarfpool.com",
         "eth-cn.dwarfpool.com","eth-cn2.dwarfpool.com",
         "eth-eu.dwarfpool.com","eth-eu1.nanopool.org",
         "eth-eu2.nanopool.org","eth-hk.dwarfpool.com",
         "eth-jp1.nanopool.org","eth-ru.dwarfpool.com",
         "eth-ru2.dwarfpool.com","eth-sg.dwarfpool.com",
         "eth-us-east1.nanopool.org","eth-us-west1.nanopool.org",
         "eth-us.dwarfpool.com","eth-us2.dwarfpool.com",
         "eu.stratum.slushpool.com","eu1.ethermine.org",
         "eu1.ethpool.org","fr.minexmr.com",
         "mine.moneropool.com","mine.xmrpool.net",
         "pool.minexmr.com","pool.monero.hashvault.pro",
         "pool.supportxmr.com","sg.minexmr.com",
         "sg.stratum.slushpool.com","stratum-eth.antpool.com",
         "stratum-ltc.antpool.com","stratum-zec.antpool.com",
         "stratum.antpool.com","us-east.stratum.slushpool.com",
         "us1.ethermine.org","us1.ethpool.org",
         "us2.ethermine.org","us2.ethpool.org",
         "xmr-asia1.nanopool.org","xmr-au1.nanopool.org",
         "xmr-eu1.nanopool.org","xmr-eu2.nanopool.org",
         "xmr-jp1.nanopool.org","xmr-us-east1.nanopool.org",
         "xmr-us-west1.nanopool.org","xmr.crypto-pool.fr",
         "xmr.pool.minergate.com", "rx.unmineable.com",
         "ss.antpool.com","dash.antpool.com",
         "eth.antpool.com","zec.antpool.com",
         "xmc.antpool.com","btm.antpool.com",
         "stratum-dash.antpool.com","stratum-xmc.antpool.com",
         "stratum-btm.antpool.com"
      ]

     - list: https_miner_domains
       items: [
        "ca.minexmr.com",
        "cn.stratum.slushpool.com",
        "de.minexmr.com",
        "fr.minexmr.com",
        "mine.moneropool.com",
        "mine.xmrpool.net",
        "pool.minexmr.com",
        "sg.minexmr.com",
        "stratum-eth.antpool.com",
        "stratum-ltc.antpool.com",
        "stratum-zec.antpool.com",
        "stratum.antpool.com",
        "xmr.crypto-pool.fr",
        "ss.antpool.com",
        "stratum-dash.antpool.com",
        "stratum-xmc.antpool.com",
        "stratum-btm.antpool.com",
        "btm.antpool.com"
      ]

     - list: http_miner_domains
       items: [
        "ca.minexmr.com",
        "de.minexmr.com",
        "fr.minexmr.com",
        "mine.moneropool.com",
        "mine.xmrpool.net",
        "pool.minexmr.com",
        "sg.minexmr.com",
        "xmr.crypto-pool.fr"
      ]

    # Add rule based on crypto mining IOCs
    
    - macro: minerpool_https
      condition: (fd.sport="443" and fd.sip.name in (https_miner_domains))

    - macro: minerpool_http
      condition: (fd.sport="80" and fd.sip.name in (http_miner_domains))

    - macro: minerpool_other
      condition: (fd.sport in (miner_ports) and fd.sip.name in (miner_domains))

    - macro: net_miner_pool
      condition: (evt.type in (sendto, sendmsg, connect) and evt.dir=< 
        and (fd.net != "127.0.0.0/8" and not fd.snet in (rfc_1918_addresses)) 
        and ((minerpool_http) or (minerpool_https) or (minerpool_other)))

    - macro: trusted_images_query_miner_domain_dns
      condition: (container.image.repository in (falco_containers))

    # The rule is disabled by default. It's enabled in this case
    # Note: falco will send DNS request to resolve miner pool domain which may trigger alerts in your environment.
    - rule: Detect outbound connections to common miner pool ports
      desc: Miners typically connect to miner pools on common ports.
      condition: net_miner_pool and not trusted_images_query_miner_domain_dns
      enabled: true
      output: Outbound connection to IP/Port flagged by https://cryptoioc.ch 
      (command=%proc.cmdline pid=%proc.pid port=%fd.rport ip=%fd.rip container=%container.info image=%container.image.repository)
      priority: CRITICAL
      tags: [host, container, network, mitre_execution, T1496]
```

Confirm all changes were formatted correctly:
```
cat custom-rules.yaml
```

So next step is to use the custom-rules.yaml file for installing the Falco Helm chart.
```
helm install falco -f custom-rules.yaml falcosecurity/falco
```

And we will see in our logs something like:
```
Mon Jan 30 10:56:26 2023: Loading rules from file /etc/falco/rules.d/rules-mining.yaml:
```
