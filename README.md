# kind-k8s-multus-example


CLUSTER=demo

for i in  $CLUSTER-worker $CLUSTER-worker2 $CLUSTER-control-plane
do
docker stop $i
docker network connect testnet $i
docker start $i
done

cilium install

##install multus plugin
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
watch "kubectl get pods --all-namespaces | grep -i multus"



cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth1@if30",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "172.17.9.0/24",
        "rangeStart": "172.17.9.240",
        "rangeEnd": "172.17.9.250",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "172.17.9.1"
      }
    }'
EOF

cat <<EOF | kubectl delete -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ipvlan-def
spec:
  config: '{
      "cniVersion": "0.3.1",
      "type": "ipvlan",
      "master": "eth1@if30",
      "mode": "l2",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.200.0/24",
        "rangeStart": "192.168.200.201",
        "rangeEnd": "192.168.200.205",
        "gateway": "192.168.200.1"
      }
    }'
EOF

cat <<EOF | kubectl apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "172.19.9.0/24",
        "rangeStart": "172.19.9.240",
        "rangeEnd": "172.19.9.250",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "172.19.9.1"
      }
    }'
EOF

##create two pods
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: net-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: netshoot-pod
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
---
apiVersion: v1
kind: Pod
metadata:
  name: net-pod2
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: netshoot-pod
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF

##verify
kubectl exec -it net-pod -- ping -c 2 -I net1 172.19.9.240

##issue solved
git clone https://github.com/containernetworking/plugins.git
cd plugins
./build_linux.sh
docker cp macvlan demo-worker:/opt/cni/bin
