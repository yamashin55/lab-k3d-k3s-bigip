# lab-k3d-k3s-bigip

## Architecture
![Architecture](./images/architecture.jpg)  

## Install docker on Ubuntu.
```
sudo apt-get update

sudo apt-get -y install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update -y

sudo apt-get install docker-ce=5:19.03.15~3-0~ubuntu-focal docker-ce-cli=5:19.03.15~3-0~ubuntu-focal -y

fbchan@sky:~$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
fbchan@sky:~$

sudo systemctl enable --now docker.service
```

## Install Helm on ubuntu
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Install Calico
```
curl -O -L https://github.com/projectcalico/calicoctl/releases/download/v3.15.0/calicoctl
chmod u+x calicoctl
sudo mv calicoctl /usr/local/bin/
```

## Install kubectl on Unbutu.
```
curl -LO https://dl.k8s.io/release/v1.19.9/bin/linux/amd64/kubectl
chmod u+x kubectl
sudo mv kubectl /usr/local/bin
```

## Install other tools.
```
sudo apt install jq -y
sudo apt install net-tools -y
```

## Install k9s on ubuntu.
```
wget https://github.com/derailed/k9s/releases/download/v0.24.2/k9s_Linux_x86_64.tar.gz
tar zxvf k9s_Linux_x86_64.tar.gz
sudo mv k9s /usr/local/bin/
```

## Disable and remove FW on ubuntu.
```
sudo ufw disable
sudo apt-get remove ufw -y
```

## Install k3s on Ubuntu.
```
wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | TAG=v4.2.0 bash
```

# Create K3s Cluster
```
k3d cluster create cpaas1 --image docker.io/rancher/k3s:v1.19.9-k3s1 \
--k3s-server-arg "--disable=servicelb" \
--k3s-server-arg "--disable=traefik" \
--k3s-server-arg --tls-san="10.42.0.10" \
--k3s-server-arg --tls-san="k3s.yshin.work" \
--k3s-server-arg '--flannel-backend=none' \
--volume "$(pwd)/calico-k3d.yaml:/var/lib/rancher/k3s/server/manifests/calico.yaml" \
--no-lb --servers 1 --agents 3
```
## kubectl get pods
```
kubectl get pod -A
```

## Setup calico on kubernetes
```
sudo mkdir /etc/calico
sudo vi /etc/calico/calicoctl.cfg

Content of calicoctl.cfg. (replace /home/xxxx/.kube/config with the location of you kubeconfig file)
---------------------------------------
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/home/ubuntu/.kube/config"
--------------------------------------

sudo calicoctl create -f 01-bgpconfig.yml
sudo calicoctl create -f 02-bgp-peer.yml
sudo calicoctl get node -o wide
```

---

## login bigip
```
ssh root@10.42.0.11  (default password is "default")
```

## bigip license activate.
```
tmsh modify auth password-policy policy-enforcement disabled
tmsh modify auth password admin 
tmsh modify auth password root 
tmsh modify sys global-settings mgmt-dhcp disabled 
tmsh create sys management-ip 1.1.1.1/32 
tmsh create net vlan internal interfaces add { 1.1 } 
tmsh create net self 10.42.0.11/16 vlan internal allow-service default 
tmsh create net route default 10.42.0.254
tmsh create net route default gw 10.42.0.254 
tmsh modify cli global-settings service number 
tmsh save sys config 
dig +nocookie www.google.com
SOAPLicenseClient --basekey CEDMQ-QUVEO-WOBRE-VYSMA-PQFVBCW
grep nw_routing_bgp bigip.license 
```


## bigip setup.
```
#https://github.com/F5Networks/f5-appsvcs-extension/releases/latest
curl -LO https://github.com/F5Networks/f5-appsvcs-extension/releases/download/v3.29.0/f5-appsvcs-3.29.0-3.noarch.rpm
vi install-rpm.sh
---------------------
#!/bin/bash
# how to use :
# chmod +x install-rpm.sh
# ./install-rpm.sh <IP address of BIG-IP> <username>:<password> <path to RPM>

set -e

if [ -z "$1" ]; then
    echo "Target machine is required for installation."
    exit 0
fi

if [ -z "$2" ]; then
    echo "Credentials [username:password] for target machine are required for installation."
    exit 0
fi

if [ -z "$3" ]; then
    echo "File path to RPM is required for installation."
    exit 0
fi

TARGET="$1"
CREDS="$2"
TARGET_RPM="$3"
RPM_NAME=$(basename $TARGET_RPM)
CURL_FLAGS="--silent --write-out \n --insecure -u $CREDS"

poll_task () {
    STATUS="STARTED"
    while [ $STATUS != "FINISHED" ]; do
        sleep 1
        RESULT=$(curl ${CURL_FLAGS} "https://$TARGET/mgmt/shared/iapp/package-management-tasks/$1")
        STATUS=$(echo $RESULT | jq -r .status)
        if [ $STATUS = "FAILED" ]; then
            echo "Failed to" $(echo $RESULT | jq -r .operation) "package:" \
                $(echo $RESULT | jq -r .errorMessage)
            exit 1
        fi
    done
}

#Get list of existing f5-appsvcs packages on target
TASK=$(curl $CURL_FLAGS -H "Content-Type: application/json" \
    -X POST https://$TARGET/mgmt/shared/iapp/package-management-tasks -d "{operation: 'QUERY'}")
poll_task $(echo $TASK | jq -r .id)
AS3RPMS=$(echo $RESULT | jq -r '.queryResponse[].packageName | select(. | startswith("f5-appsvcs"))')

#Uninstall existing f5-appsvcs packages on target
for PKG in $AS3RPMS; do
    echo "Uninstalling $PKG on $TARGET"
    DATA="{\"operation\":\"UNINSTALL\",\"packageName\":\"$PKG\"}"
    TASK=$(curl ${CURL_FLAGS} "https://$TARGET/mgmt/shared/iapp/package-management-tasks" \
        --data $DATA -H "Origin: https://$TARGET" -H "Content-Type: application/json;charset=UTF-8")
    poll_task $(echo $TASK | jq -r .id)
done

#Upload new f5-appsvcs RPM to target
echo "Uploading RPM to https://$TARGET/mgmt/shared/file-transfer/uploads/$RPM_NAME"
LEN=$(wc -c $TARGET_RPM | awk 'NR==1{print $1}')
RANGE_SIZE=5000000
CHUNKS=$(( $LEN / $RANGE_SIZE))
for i in $(seq 0 $CHUNKS); do
    START=$(( $i * $RANGE_SIZE))
    END=$(( $START + $RANGE_SIZE))
    END=$(( $LEN < $END ? $LEN : $END))
    OFFSET=$(( $START + 1))
    curl ${CURL_FLAGS} -o /dev/null --write-out "" \
        https://$TARGET/mgmt/shared/file-transfer/uploads/$RPM_NAME \
        --data-binary @<(tail -c +$OFFSET $TARGET_RPM) \
        -H "Content-Type: application/octet-stream" \
        -H "Content-Range: $START-$(( $END - 1))/$LEN" \
        -H "Content-Length: $(( $END - $START ))" \
        -H "Connection: keep-alive"
done

#Install f5-appsvcs on target
echo "Installing $RPM_NAME on $TARGET"
DATA="{\"operation\":\"INSTALL\",\"packageFilePath\":\"/var/config/rest/downloads/$RPM_NAME\"}"
TASK=$(curl ${CURL_FLAGS} "https://$TARGET/mgmt/shared/iapp/package-management-tasks" \
    --data $DATA -H "Origin: https://$TARGET" -H "Content-Type: application/json;charset=UTF-8")
poll_task $(echo $TASK | jq -r .id)

echo "Waiting for /info endpoint to be available"
until curl ${CURL_FLAGS} -o /dev/null --write-out "" --fail --silent \
    "https://$TARGET/mgmt/shared/appsvcs/info"; do
    sleep 1
done

echo "Installed $RPM_NAME on $TARGET"

exit 0
---------------------

chmod +x install-rpm.sh
./install-rpm.sh 10.42.0.11 admin:admin ./f5-appsvcs-3.29.0-3.noarch.rpm 
curl -ks -u admin:admin https://10.42.0.11/mgmt/shared/appsvcs/info |jq .
```



## BGP setup on bigip.
```
tmsh modify net route-domain 0 routing-protocol add { BGP } 
tmsh create auth partition kubernetes

imish
enable
configure terminal
router bgp 64512
bgp graceful-restart restart-time 120
neighbor calico-k8s peer-group
neighbor calico-k8s remote-as 64512
neighbor 172.18.0.2 peer-group calico-k8s
neighbor 172.18.0.3 peer-group calico-k8s
neighbor 172.18.0.4 peer-group calico-k8s
neighbor 172.18.0.5 peer-group calico-k8s
write
end
show running-config
exit
```


## Route add on bigip.
```
tmsh create net route ubuntu-k3d network 172.18.0.0/16 gw 10.42.0.10
```

## Route add on ubuntu. 
```
sudo ip route add 10.53.86.64/26 via 172.18.0.3
sudo ip route add 10.53.68.192/26 via 172.18.0.4
sudo ip route add 10.53.115.0/26 via 172.18.0.5
sudo ip route add 10.53.194.192/26 via 172.18.0.2

```
