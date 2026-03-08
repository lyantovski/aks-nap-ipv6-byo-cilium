# Network setup
LOCATION="swedencentral"
RG="customer-cilium-rg"
VNET="customer-cilium-vnet"
SUBNET="customer-cilium-snet-aks"
VNET_V4="172.16.0.0/16"
VNET_V6="fdf0:1e2d:3c4b::/48"   
SUBNET_V4="172.16.1.0/24"
SUBNET_V6="fdf0:1e2d:3c4b::/64"

# Create Resource Group
az group create --name $RG --location $LOCATION

# Create VNet with IPv4 and IPv6 prefixes
az network vnet create \
    --resource-group $RG \
    --name $VNET \
    --address-prefixes $VNET_V4 $VNET_V6

# Create a Dual-Stack Subnet
az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET \
    --name $SUBNET \
    --address-prefixes $SUBNET_V4 $SUBNET_V6

#subnet id
SUBNET_ID=$(az network vnet subnet show -g "$RG" --vnet-name "$VNET" -n "$SUBNET" --query id -o tsv)
echo "$SUBNET_ID"

# create managed identity
IDENTITY_NAME="customer-cilium-aks-mi"
az identity create -g "$RG" -n "$IDENTITY_NAME" -l "$LOCATION"

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
IDENTITY_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME"
IDENTITY_PRINCIPAL_ID=$(az identity show --resource-group $RG --name $IDENTITY_NAME --query principalId -o tsv)

Verify: 
echo $IDENTITY_ID
echo $IDENTITY_PRINCIPAL_ID
echo $SUBSCRIPTION_ID

az role assignment create --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$VNET" \
--role "Network Contributor" \
--assignee $IDENTITY_PRINCIPAL_ID

#AKS Cluster Creation
CLUSTER_NAME="customer-cilium-aks"
POD_CIDR_V4="10.244.0.0/16"
SVC_CIDR_V4="10.0.0.0/16"
DNS_IP_V4="10.0.0.10"

az aks create \
    --resource-group $RG \
    --name $CLUSTER_NAME \
    --location $LOCATION \
    --vnet-subnet-id "$SUBNET_ID" \
    --assign-identity "$IDENTITY_ID" \
    --network-plugin none \
    --enable-managed-identity \
    --ip-families ipv4 \
    --node-provisioning-mode Auto \
    --nodepool-name "system" \
    --pod-cidrs "$POD_CIDR_V4" \
    --service-cidrs "$SVC_CIDR_V4" \
    --dns-service-ip "$DNS_IP_V4" \
    --node-count 1 \
    --generate-ssh-keys

az aks get-credentials --resource-group $RG --name $CLUSTER_NAME --overwrite-existing

# You will get status of Nodes - NotReady as expected
$ kubectl get nodes
NAME                             STATUS     ROLES    AGE   VERSION
aks-system-40454228-vmss000000   NotReady   <none>   29m   v1.33.7
aks-system-surge-bjb8l           NotReady   <none>   25m   v1.33.7
aks-system-surge-gxcl9           NotReady   <none>   25m   v1.33.7
aks-system-surge-w4h86           NotReady   <none>   25m   v1.33.7

az aks nodepool update \
  -g "$RG" \
  -n "system" \
  --cluster-name "$CLUSTER_NAME" \
  --node-taints CriticalAddonsOnly=true:NoSchedule

Docs ref:
https://github.com/cilium/cilium-cli/releases/download/v0.19.2/cilium-windows-amd64.zip
https://github.com/cilium/cilium/blob/main/install/kubernetes/cilium/README.md
https://github.com/cilium/cilium/tree/main/pkg/k8s/apis/cilium.io/client/crds/v2 

#Install Cilium
$ helm repo add cilium https://helm.cilium.io/
$ helm repo update

API_SERVER_IP=$(kubectl get endpoints kubernetes -o jsonpath='{.subsets[0].addresses[0].ip}')
echo "API Server IP is: $API_SERVER_IP"

helm install cilium cilium/cilium --version 1.19.1 \
  --namespace kube-system \
  --set ipv4.enabled=true \
  --set ipv6.enabled=true \
  --set enableIPv6Masquerade=true \
  --set kubeProxyReplacement=true \
  --set dnsproxy.enabled=true \
  --set localRedirectPolicies.enabled=true \
  --set k8sServiceHost=$API_SERVER_IP \
  --set k8sServicePort=443 \
  --set aksbyocni.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.relay.enabled=true \
  --set operator.replicas=2

$ kubectl get nodes -o wide
NAME                             STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-default-grbj5                Ready    <none>   5m50s   v1.33.7   172.16.1.8    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-system-40454228-vmss000000   Ready    <none>   53m     v1.33.7   172.16.1.4    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-system-surge-w4h86           Ready    <none>   49m     v1.33.7   172.16.1.7    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2


#Manual IPv6 assignment to node - for tests purpose only. For real scenario we will use daemonset in ../ipv6-nap-manager.
#Assign IPv6 public IP to node
VM="aks-default-grbj5"   # one of AKS NAP nodes above
MC_RG="MC_customer-cilium-rg_customer-cilium-aks_swedencentral"
PIP6_NAME="${VM}-pip6"
IP_CONFIG_NAME="ipconfig6"
 
NIC_ID=$(az vm show -g "$MC_RG" -n "$VM" --query "networkProfile.networkInterfaces[0].id" -o tsv)
NIC_NAME=$(basename "$NIC_ID")
echo $NIC_NAME


# assign ipv6 public ip to nic
az network public-ip create \
  -g "$MC_RG" -n "$PIP6_NAME" \
  --sku Standard \
  --version IPv6 \
  --allocation-method Static

Copy the ipv6 address from output - for example: 2603:1020:1001:4::144

# get subnet id
SUBNET_ID=$(az network nic show -g "$MC_RG" -n "$NIC_NAME" --query "ipConfigurations[0].subnet.id" -o tsv)
echo $SUBNET_ID

az network nic ip-config create \
  -g "$MC_RG" --nic-name "$NIC_NAME" \
  -n "$IP_CONFIG_NAME" \
  --subnet "$SUBNET_ID" \
  --private-ip-address-version IPv6 \
  --public-ip-address "$PIP6_NAME"

#Lets look on AKS Node aks-default-grbj5 (ipconfig6):
Public IP address: 2603:1020:1001:4::144
Private IP address: fdf0:1e2d:3c4b::4

$ kubectl get nodes -o wide
NAME                             STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP             OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-default-bsqxr                Ready    <none>   172m    v1.33.7   172.16.1.10   2603:1020:1001:4::144   Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-default-hvvft                Ready    <none>   173m    v1.33.7   172.16.1.6    <none>                  Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2
aks-system-40454228-vmss000000   Ready    <none>   5h19m   v1.33.7   172.16.1.4    <none>                  Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.30-2

#In case CRD CiliumLocalRedirectPolicy not exist (although we enabled in installation step):
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/refs/heads/main/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumlocalredirectpolicies.yaml

#Note provided cilium-app.yaml and cilium-dns-policy.yaml separately for more easy usage.

kubectl apply -f - <<EOF
apiVersion: "cilium.io/v2"
kind: CiliumLocalRedirectPolicy
metadata:
  name: "dns-hijack-global"
  namespace: kube-system
spec:
  redirectFrontend:
    # This matches the destination IP currently in your pods' /etc/resolv.conf
    addressMatcher:
      ip: "168.63.129.16"
      toPorts:
        - port: "53"
          protocol: UDP
          name: dns-udp
        - port: "53"
          protocol: TCP
          name: dns-tcp
  redirectBackend:
    # This tells Cilium to send that traffic to the CoreDNS pod on the SAME node
    localEndpointSelector:
      matchLabels:
        k8s-app: kube-dns
    toPorts:
      - port: "53"
        protocol: UDP
        name: dns-udp
      - port: "53"
        protocol: TCP
        name: dns-tcp
EOF


# application for tests
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hostnet
  namespace: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-hostnet
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx-hostnet
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
    # Force Anti-affinity
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: nginx-hostnet
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 8080
              hostPort: 8080
              protocol: TCP
          command: ["/bin/sh","-c"]
          args:
            - |
              cat >/etc/nginx/conf.d/default.conf <<'EOF'
              server {
                listen 8080;
                listen [::]:8080;
                server_name _;
                location / {
                  return 200 "nginx hostNetwork OK on $hostname\n";
                }
              }
              EOF
              nginx -g 'daemon off;'
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 128Mi
--- 
apiVersion: v1
kind: Service
metadata:
  name: nginx-hostnet
  namespace: nginx-demo
  labels:
    app: nginx-hostnet
spec:
  selector:
    app: nginx-hostnet
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP 
EOF


Test connectivity from browser: https://8gwifi.org/curlfunctions.jsp
curl -6 http://[2603:1020:1001:4::144]:8080

Test localy:
/ # curl -6 http://[2603:1020:1001:4::144]:8080
nginx hostNetwork OK on aks-default-grbj5
