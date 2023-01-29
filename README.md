# Falco Cryptomining Lab - [CNCF]
This session will explain some of the intricacies of crypto-mining in Kubernetes and how crypto-jackers compromise containerized workloads at runtime. Cryptojacking is not limited to containers and hosts.

Trust the ```falcosecurity GPG``` key, configure the ```apt``` repository, and update the package list:
```
curl -s https://falco.org/repo/falcosecurity-packages.asc | apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | tee -a /etc/apt/sources.list.d/falcosecurity.list
apt-get update -y
```
Install kernel headers:
```
apt-get -y install linux-headers-$(uname -r)
```
Install Falco:
```
apt-get install -y falco
```
