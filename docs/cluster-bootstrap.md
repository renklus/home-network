# Install cluster from scratch
## One time setup of management workstation
### Install kubectl
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
```sh
curl -LO https://dl.k8s.io/release/v1.37.0/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
# verify with
kubectl version --client
```

### Configure kubeconfig
Only works after k3s is installed \
https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/ \
https://www.reddit.com/r/kubernetes/comments/1dovs69/how_do_i_manage_multiple_kubeconfig_files_in_my/
```sh
ssh mgmt@k8s01d.dev.renklus.ch 'sudo -S cat /etc/rancher/k3s/k3s.yaml' > ~/.kube/config-k8s01d
# chmod to 660 if it does not work
chmod 600 ~/.kube/config-k8s01d
# set server address (for certificate check to work only use server name, not FQDN)
nano ~/.kube/config-k8s01d
echo "alias k8s01d='kubectl --kubeconfig ~/.kube/config-k8s01d'" >> ~/.bashrc
source ~/.bashrc
```

### Install Helm
https://helm.sh/docs/intro/install/
```sh
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm repo update
```

## Setup first node in rancher management cluster
```sh
sudo apt-get update
sudo apt-get install -y curl
# better to not set --cluster-domain
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL="stable" INSTALL_K3S_VERSION="v1.37.0+k3s1" INSTALL_K3S_EXEC='--disable="servicelb"' sh -s - server --cluster-init
sudo k3s kubectl get nodes
sudo k3s kubectl get pods -A
```

Leave SWAP enabled !!! Disabling SWAP in k8s is only recommended because the pod scheduler makes mistakes in selecting the correct node when SWAP is enabled. \
Discussion: https://github.com/kubernetes/kubernetes/issues/53533 \
Instead set memory requirements for pods to disable SWAP usage for pods: https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-355526636 \
List SWAP:
```sh
systemctl list-unit-files | grep swap
# DO NOT RUN `systemctl mask` disables swap
# sudo systemctl mask "dev-disk-by\x2duuid-3361cd51\x2d9092\x2d4535\x2da094\x2d3ffe61a11b55.swap"
# after reboot verify with
free
sudo k3s check-config
```


## Add additional server node to rancher management cluster
```sh
# On first node
sudo cat /var/lib/rancher/k3s/server/token
# On additional node
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL="stable" INSTALL_K3S_VERSION="v1.37.0+k3s1" K3S_TOKEN=<Secret> INSTALL_K3S_EXEC='--disable="servicelb"' sh -s - server --server https://<ip or hostname of server1>:6443
```

## Add additional worker node to rancher management cluster
Probably not required \
Should work similar to chapter above, but there is a lower privilege token under a different path for just adding agents and the install command is slightly different so it configures the new node as just a worker

## External setup
- In LAN: Query forward domain rancher.k8s.renklus.ch to 10.1.0.245 (IP from coredns.yaml)
- In LAN: Query forward domain prod.k8s.renklus.ch to 10.1.1.254 (IP from coredns-expose.yaml)
- Not anymore: In LAN: Query forward domain ingress.rancher.k8s.renklus.ch to 10.1.0.244 (IP from traefik.yaml)
- Not anymore: In LAN: Query forward domain ingress.prod.k8s.renklus.ch to 10.1.1.253 (IP from traefik.yaml)
- In LAN: Setup Destination NAT on WAN interface for WAN IP:80 to haproxy.haproxy.rancher.k8s.renklus.ch:8000 (For Let's Encrypt ACME check)
- Not anymore: In LAN: Setup Destination NAT on LAN interface for WAN IP:80 to haproxy.haproxy.rancher.k8s.renklus.ch:8000 (For Cert Manager ACME self check)
- Not anymore: In LAN: Setup Source NAT
- In Cloudflare: forward domain *.k8s.renklus.ch to public WAN IP

## Install applications
```sh
helm install argo-cd argo-cd/argo-cd --version 10.9.2 --kubeconfig ~/.kube/config-k8s01d --namespace argocd --create-namespace -f https://raw.githubusercontent.com/renklus/home-network/refs/heads/main/k8s-rancher/apps/argocd/values.yaml
k8s01d apply --server-side --force-conflicts -f https://raw.githubusercontent.com/renklus/home-network/refs/heads/main/k8s-rancher/apps/app-of-apps.yaml
# k8s01d -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
k8s01d -n argocd get secret argocd-initial-admin-secret -o go-template='{{.data.password|base64decode}}{{"\n"}}'
```
- Then wait 15 minutes
- log in to https://argo.rancher.k8s.renklus.ch with 'admin' and the secret from before

## Add additional worker/server node to any cluster (except rancher management cluster)
- log in to rancher.ingress.rancher.k8s.renklus.ch `k8s01d -n cattle-system get secret bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'`
- Navigate to the cluster and copy the node-join command from the web interface. Remember to specify a node name under advanced settings
  - If the command is not displayed correctly delete cluster from rancher web UI or reboot rancher host OS. Command will be correct afterwards.