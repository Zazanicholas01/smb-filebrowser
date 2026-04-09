## **Initial Setup**

#### ***Prepare Ubuntu machine***
Inside Ubuntu VM or WSL, update packages:
```bash
sudo apt update
sudo apt -y full-upgrade
sudo apt -y install curl ca-certificates gnupg apt-transport-https cifs-utils smbclient
```
---
#### ***Install K3S Cilium-optimized***

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='server --write-kubeconfig-mode 644 --flannel-backend=none --disable-network-policy --disable=traefik --disable=servicelb' sh -
```
Verify installation:
- Export Kubeconfig
- Get nodes & pods (Timeout expected)

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes -o wide
kubectl get pods -A
```
---

#### ***Install Cilium through Helm***

Install Helm, if not already installed:
```bash
sudo snap install helm --classic
helm version
```
Then install Cilium:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" \
  --set operator.replicas=1
```

Wait and verify:
```bash
kubectl -n kube-system rollout status ds/cilium --timeout=300s
kubectl -n kube-system rollout status deploy/cilium-operator --timeout=300s
kubectl get nodes
kubectl get pods -A
```
---

#### ***Create SMB Storage on Powershell***

```powershell
New-Item -ItemType Directory -Force C:\k8s-smb-data
New-SmbShare -Name k8sdata -Path C:\k8s-smb-data -FullAccess $env:USERNAME
Enable-NetFirewallRule -DisplayGroup "Condivisione file e stampanti"
```

#### ***Install Samba CSI Driver***

```bash
helm repo add csi-driver-smb https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
helm repo update

helm upgrade --install csi-driver-smb csi-driver-smb/csi-driver-smb \
  --namespace kube-system \
  --version v1.20.1
```

#### ***Create resources to test file browser setup***

SECRET - DATA WITH THIS FORMAT REQUIRED
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: smbcreds
  namespace: kube-system
type: Opaque
stringData:
  username: "k8suser"
  password: "Nicholas01!"
  domain: "LAPTOP-IR4QNK4V"
```

STORAGE CLASS
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: smb-filebrowser
provisioner: smb.csi.k8s.io
parameters:
  source: "//172.31.64.1/k8sdata"
  csi.storage.k8s.io/provisioner-secret-name: smbcreds
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
  csi.storage.k8s.io/node-stage-secret-name: smbcreds
  csi.storage.k8s.io/node-stage-secret-namespace: kube-system
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - dir_mode=0770
  - file_mode=0660
  - uid=1000
  - gid=1000
  - vers=3.0
```

NAMESPACE
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: filebrowser
```

PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: filebrowser-pvc
  namespace: filebrowser
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: smb-filebrowser
  resources:
    requests:
      storage: 10Gi
```

If PVC is bound and not Pending, SMB storage setup is finished.

#### ***Known problem with user setup***

The key line is:
```bash
mount error(13): Permission denied
```
The driver can reach the Samba share but Windows rejects credentials or permissions for the CSI mount.
- Clean fix -> Create a dedicated local Windows user just for SMB.

Run Powershell as Administrator.
```powershell
$pw = Read-Host -AsSecureString "Password for k8suser"
New-LocalUser -Name "k8suser" -Password $pw -FullName "k8suser" -Description "SMB user for k3s"
Add-LocalGroupMember -Group "Users" -Member "k8suser"
Grant-SmbShareAccess -Name "k8sdata" -AccountName "k8suser" -AccessRight Full -Force
icacls C:\k8s-smb-data /grant "k8suser:(OI)(CI)F"
```
Verify from Ubuntu:
```bash
smbclient //172.31.64.1/k8sdata -U 'LAPTOP-IR4QNK4V\k8suser'
```
Then re-apply secret and delete / recreate the PVC:
```bash
k3s kubectl apply -f smb-filebrowser.yaml
k3s kubectl -n filebrowser delete pvc filebrowser-pvc
k3s kubectl apply -f smb-filebrowser.yaml
k3s kubectl -n filebrowser describe pvc filebrowser-pvc
```

#### ***Filebrowser Deployment***

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: filebrowser
  namespace: filebrowser
spec:
  replicas: 1
  selector:
    matchLabels:
      app: filebrowser
  template:
    metadata:
      labels:
        app: filebrowser
    spec:
      securityContext:
        fsGroup: 1000
      containers:
        - name: filebrowser
          image: filebrowser/filebrowser
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /srv
              subPath: srv
            - name: data
              mountPath: /database
              subPath: database
            - name: data
              mountPath: /config
              subPath: config
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: filebrowser-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: filebrowser
  namespace: filebrowser
spec:
  selector:
    app: filebrowser
  ports:
    - port: 80
      targetPort: 80
```
Once working, get the **credentials** to access `Filebrowser UI` by checking the logs:
```bash
k3s kubectl -n filebrowser logs deploy/filebrowser
```
Check if working by forwarding the port first:
```bash
k3s kubectl -n filebrowser port-forward svc/filebrowser 8080:80
```
Then access with: `http://localhost:8080`

---
#### ***Nginx Gateway API***

Install through Helm:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
k3s kubectl config current-context
k3s kubectl get nodes
helm --kubeconfig /etc/rancher/k3s/k3s.yaml install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace \
  -n nginx-gateway \
  --set nginx.service.type=NodePort
```
Verify:
```bash
k3s kubectl -n nginx-gateway get pods
k3s kubectl -n nginx-gateway get svc
```
Create Gateway resource and HTTP Route to start:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: filebrowser-gateway
  namespace: filebrowser
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: filebrowser-route
  namespace: filebrowser
spec:
  parentRefs:
    - name: filebrowser-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: filebrowser
          port: 80
```
Apply and veriy:
```bash
k3s kubectl apply -f gateway-filebrowser.yaml

k3s kubectl -n filebrowser get gateway
k3s kubectl -n filebrowser get httproute
```
Access it with if port forwarding: `http://localhost:<nodeport>`

Access through the NodePort by getting the Node IP:
```bash
k3s kubectl get nodes -o wide
http://<node_IP>:<nodeport>
```
---
#### ***Configure Cilium for L2 Load balancing & Advertisement - OPTIONAL***

```bash
helm --kubeconfig /etc/rancher/k3s/k3s.yaml upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set l2announcements.enabled=true

k3s kubectl -n kube-system rollout status ds/cilium --timeout=300s
k3s kubectl -n kube-system rollout status deploy/cilium-operator --timeout=300s
```

Create pool and policy:
```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: lan-pool
spec:
  blocks:
    - start: 10.35.75.240
      stop: 10.35.75.245
---
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: lan-l2
spec:
  loadBalancerIPs: true
  externalIPs: true
  interfaces:
    - eth0
```
Apply:
```bash
k3s kubectl apply -f cilium-lb.yaml
```
To enable Load Balancer services through Nginx Gateway by using Cilium, 
upgrade Gateway's Helm chart:
```bash
helm --kubeconfig /etc/rancher/k3s/k3s.yaml upgrade ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --namespace nginx-gateway \
  --reuse-values \
  --set nginx.service.type=LoadBalancer
```
Wait for an External IP from the IP pool of Cilium:
```bash
k3s kubectl -n nginx-gateway rollout status deploy/ngf-nginx-gateway-fabric --timeout=300s
k3s kubectl -n filebrowser get svc filebrowser-gateway-nginx -w
```
---

#### ***TLS and Certificates Setup***

Create OpenSSL key pair for the domain:
```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout files.home.nipo.key \
  -out files.home.nipo.crt \
  -subj "/CN=files.home.nipo" \
  -addext "subjectAltName=DNS:files.home.nipo"
```
Create Kubernetes TLS Secret to use those keys:
```bash
k3s kubectl -n filebrowser delete secret filebrowser-tls --ignore-not-found
k3s kubectl -n filebrowser create secret tls filebrowser-tls \
  --cert=files.home.nipo.crt \
  --key=files.home.nipo.key
```
Update Gateway to add an HTTPS listener:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: filebrowser-gateway
  namespace: filebrowser
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: files.home.nipo
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      protocol: HTTPS
      port: 443
      hostname: files.home.nipo
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: filebrowser-tls
      allowedRoutes:
        namespaces:
          from: Same
```
And also update HTTP Route to bind to HTTPS and new hostname:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: filebrowser-route
  namespace: filebrowser
spec:
  parentRefs:
    - name: filebrowser-gateway
      sectionName: https
  hostnames:
    - files.home.nipo
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: filebrowser
          port: 80
```
Apply:
```bash
k3s kubectl apply -f gateway-filebrowser.yaml
```
Then add to Windows hosts: `C:\Windows\System32\drivers\etc\hosts`:
```
<Load_Balancer_IP>  files.home.nipo
```
Then open: `https://files.home.nipo`

On Linux VM, this should be enough, but on WSL not.

---

#### ***WSL Problem Troubleshooting***

Now there is a real blocker with a well-known WSL2 network limitation, not a Cilium bad config.
Why timeout:
- Load Balancer IP is o Wi-Fi LAN: `10.35.75.240`
- K3S Node actually lives on WSL's private NAT network: `172.31.65.221/20`
- Inside WSL it works
- From Windows/LAN, times out because WSL2 is not bridged onto the physical LAN, so it cannot properly ARP / advertise that `10.35.75.240` on the real network.

How to fix:
- Go back to nodeport on Gateway API:
```bash
helm --kubeconfig /etc/rancher/k3s/k3s.yaml upgrade ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --namespace nginx-gateway \
  --reuse-values \
  --set nginx.service.type=NodePort
```
- WSL Node IP -- `172.31.65.221`
- HTTP NodePort -- `32439`
- HTTPS NodePort -- `30135`

It still can't reach the Virtual External IP from Cilium but can proxy to node ports:
- files.home.nipo -- `127.0.0.1`
- `127.0.0.1:80` -> `172.31.65.221:<http-nodeport>`
- `127.0.0.1:443` -> `172.31.65.221:<https-nodeport>`

The Load Balancer IP is still assigned but ignored. Real traffic flows through node port proxies.

Open Powershell as Administrator:
```powershell
netsh interface portproxy add v4tov4 listenaddress=127.0.0.1 listenport=80 connectaddress=172.31.65.221 connectport=32439
netsh interface portproxy add v4tov4 listenaddress=127.0.0.1 listenport=443 connectaddress=172.31.65.221 connectport=30135
```
Open firewall for those local ports:
```powershell
netsh advfirewall firewall add rule name="WSL NGF HTTP 80" dir=in action=allow protocol=TCP localport=80
netsh advfirewall firewall add rule name="WSL NGF HTTPS 443" dir=in action=allow protocol=TCP localport=443
```
List proxies ports if needed:
```powershell
netsh interface portproxy show all
```
Then add to Windows hosts file: `C:\Windows\System32\drivers\etc\hosts`:
```
127.0.0.1  files.home.nipo
```
Flush DNS:
```
ipconfig /flushdns
```
Then access it on: `https://files.home.nipo`

#### ***Homepage Setup***

HOMEPAGE PVC + NAMESPACE
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: homepage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: homepage-pvc
  namespace: homepage
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: smb-filebrowser
  resources:
    requests:
      storage: 2Gi
```
HOMEPAGE CONFIG
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: homepage-config
  namespace: homepage
data:
  settings.yaml: |
    title: Home
    color: slate
    headerStyle: clean
  widgets.yaml: |
    - search:
        provider: google
        target: _blank
    - datetime:
        text_size: xl
        format:
          dateStyle: short
          timeStyle: short
  services.yaml: |
    - Storage:
        - File Browser:
            href: https://files.home.nipo
            description: SMB-backed file browser
            icon: sh-filebrowser
  bookmarks.yaml: |
    - Developer:
        - GitHub:
            - abbr: GH
              href: https://github.com/
  docker.yaml: ""
  kubernetes.yaml: |
    mode: cluster
  custom.css: ""
  custom.js: ""
```
HOMEPAGE DEPLOYMENT + SERVICE
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homepage
  namespace: homepage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: homepage
  template:
    metadata:
      labels:
        app: homepage
    spec:
      initContainers:
        - name: init-config
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              mkdir -p /work/logs
              cp /seed/* /work/
          volumeMounts:
            - name: config-seed
              mountPath: /seed
            - name: config-work
              mountPath: /work
      containers:
        - name: homepage
          image: ghcr.io/gethomepage/homepage:latest
          ports:
            - containerPort: 3000
          env:
            - name: HOMEPAGE_ALLOWED_HOSTS
              value: home.home.nipo
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
          volumeMounts:
            - name: config-work
              mountPath: /app/config
      volumes:
        - name: config-seed
          configMap:
            name: homepage-config
        - name: config-work
          emptyDir: {}
```
HOMEPAGE ROUTE
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: homepage-route
  namespace: homepage
spec:
  parentRefs:
    - name: filebrowser-gateway
      namespace: filebrowser
      sectionName: home-https
  hostnames:
    - home.home.nipo
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: homepage
          port: 3000
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-homepage-to-filebrowser-gateway
  namespace: filebrowser
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: homepage
  to:
    - group: gateway.networking.k8s.io
      kind: Gateway
```
Apply:
```bash
k3s kubectl apply -f homepage-storage.yaml
k3s kubectl apply -f homepage-config.yaml
k3s kubectl apply -f homepage-deploy.yaml
k3s kubectl apply -f homepage-route.yaml
```

Update Gateway to support new route:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: filebrowser-gateway
  namespace: filebrowser
spec:
  gatewayClassName: nginx
  listeners:
    ...
    - name: home-http
      protocol: HTTP
      port: 80
      hostname: home.home.nipo
      allowedRoutes:
        namespaces:
          from: All
    - name: home-https
      protocol: HTTPS
      port: 443
      hostname: home.home.nipo
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: homepage-tls
      allowedRoutes:
        namespaces:
          from: All
```
Generate certificates for new domain:
```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout home.home.nipo.key \
  -out home.home.nipo.crt \
  -subj "/CN=home.home.nipo" \
  -addext "subjectAltName=DNS:home.home.nipo"
```
Create Kubernetes TLS Secret:
```bash
k3s kubectl -n filebrowser create secret tls homepage-tls \
  --cert=home.home.nipo.crt \
  --key=home.home.nipo.key \
  --dry-run=client -o yaml | k3s kubectl apply -f -
```
Add to `hosts` Windows file:
```
127.0.0.1  home.home.nipo
```

#### ***Keycloak OIDC Authentication***

Target design:
- Homepage behind oauth2-proxy + Keycloak
- File Browser behind oauth2-proxy + Keycloak
- Shared auth cookie domain `.home.nipo`

#### ***Wireguard VPN Setup***

