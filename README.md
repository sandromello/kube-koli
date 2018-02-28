# AWS Notes - Requirements

- Install the [master IAM role](./aws/master-IAM.json)
- Install the [node IAM role](./aws/node-IAM.json)
- Configure Security Groups (nodes/master) [ TODO DOCS ]
- Network Load Balancer (NLB) with targets for each node on ports `30080` and `30443`
- Elastic Load Balancer (ELB) redirecting to port `30600` on each node (mutator/websockets)

[More info](https://docs.google.com/document/d/17d4qinC_HnIwrK0GHnRlD1FKkTNdN__VO4TH9-EzbIY/edit)

# Auth0 Notes - Requirements

- Add a Single Page Application Client (RS256)
  * Kubernetes API Server (oidc-client-id)
  * Koli Portal (SPA Client ID)
  * Add proper Callback, Logout and CORS URL's
- Add a Non Interactive Application Client (HS256)
  * Gitstep
- Add Management API with RS256 signing algorithm
  * Allow Non Interactive Client Authorize 
- Add Connection (SOCIAL) GitHub Client

> Clients must not be OIDC Conformant (advanced settings)

# Cluster Installation on AWS

- Launch EC2 instances from a [pre-defined AMI.](./aws/packer/README.md) (master/node)
- Set a AWS tag for each instance informing the cluster name `kubernetes.io/cluster/<cluster-name>=shared` (master/node)
- Configure `aws-provider` option to kubelet's (master/node)

```bash
sudo sed -e "/cadvisor-port=0/d" -i /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo bash -c "cat <<EOF > /etc/systemd/system/kubelet.service.d/20-extra-args.conf 
[Service]
Environment="KUBELET_EXTRA_ARGS=--cloud-provider=aws"
EOF"
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

- Start a master control plane (master)

```bash
NODENAME=$(hostname -f)
ADVERTISE_ADDRESS=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
# https://github.com/kubernetes/kubernetes/releases
KUBERNETES_VERSION=$(cat https://dl.k8s.io/release/stable.txt)
OIDC_CLIENT_ID=
OIDC_ISSUER_URL=https://koli.auth0.com/
CLUSTER_NAME=rhea

cat <<EOF >$HOME/master.yml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: $ADVERTISE_ADDRESS
  bindPort: 6443
authorizationModes:
- Node
- RBAC
cloudProvider: aws
certificatesDir: /etc/kubernetes/pki
apiServerCertSANs:
- k8s.$CLUSTER_NAME.kolihub.io
- api.$CLUSTER_NAME.kolihub.io
- api.kolihub.io
apiServerExtraArgs:
  insecure-port: '8080'
  oidc-client-id: $OIDC_CLIENT_ID
  oidc-groups-claim: groups
  oidc-issuer-url: $OIDC_ISSUER_URL
  oidc-username-claim: email
controllerManagerExtraArgs:
  configure-cloud-routes: 'false'
  address: 0.0.0.0
schedulerExtraArgs:
  address: 0.0.0.0
etcd:
  caFile: ""
  certFile: ""
  dataDir: /var/lib/etcd
  endpoints: null
  image: ""
  keyFile: ""
imageRepository: gcr.io/google_containers
kubernetesVersion: $KUBERNETES_VERSION
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
nodeName: $NODENAME
token: ""
tokenTTL: 24h0m0s
unifiedControlPlaneImage: ""
EOF

# Copy the command stdout that you'll need for joining your nodes
sudo kubeadm init --config master.yml
```

- Configure kubectl (optional)

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Install Calico networking (master)

```bash
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
```

## Join Nodes

- Configure kubelet with `cloud-provider` option before continuing (see above)

```bash
# This command is printed on stdout when you run `kubeadm init`
kubeadm join --token <token> <k8s-master-host>:6443 --discovery-token-ca-cert-hash sha256:<token-ca-cert-hash>

# Set the role as node
kubectl get nodes
kubectl label node ip-10-0-79-218.ec2.internal kubernetes.io/role=node
kubectl label node ip-10-0-79-218.ec2.internal node-role.kubernetes.io/node=""
```

> **TIP:** Check the [reference guide of kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/) for more information.

## Configure kubectl externally

- Associate a DNS with the master node instance (must conform with the `apiServerCertSANs` option in `master.yml`)
- Copy the contents of $HOME/.kube/config
- Merge or add to your local $HOME/.kube/config and modify the address to your assigned DNS

## Install / Configure Helm

```bash
# https://github.com/kubernetes/helm/releases
HELM_VERSION=v2.8.1
curl https://storage.googleapis.com/kubernetes-helm/helm-$HELM_VERSION-linux-amd64.tar.gz | tar -O -xzvf - linux-amd64/helm > helm && sudo mv ./helm /usr/local/bin/helm
sudo chown root: /usr/local/bin/helm && sudo chmod +x /usr/local/bin/helm

kubectl -n kube-system create sa tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```

# Platform Installation

```bash
kubectl create ns koli-system
# allow traffic between koli-system and platform namespaces
kubectl label ns koli-system kolihub.io/kong-system="true"
```

- Add default storage class

```bash
# https://kubernetes.io/docs/concepts/storage/storage-classes/#the-storageclass-resource
kubectl create -f ./manifests/k8s/default-storage-class.yml
```

- Install EFS Persistent Volumes

```bash
# IMPORTANT: Edit the address of the NFS server before creating it
kubectl apply -f ./manifests/k8s/efs-postgres-pv.yml
kubectl apply -f ./manifests/k8s/efs-gitstep.yml
```

## Install Required Secrets

```bash
# Create a random password for postgres database
kubectl create secret -n koli-system generic pg-credentials \
  --from-literal=password=$(date +%s | shasum | base64 | head -c 32 ; echo) \
  --from-literal=username=system

SECRETS_PATH=

# auth0 credentials
AUTH0_SINGLE_PAGE_APP_CRT_PATH=$SECRETS_PATH/single-page-app-auth0.crt
AUTH0_SINGLE_PAGE_APP_CLIENT_ID=$(cat $SECRETS_PATH/single-page-app-client-id.key)
AUTH0_SINGLE_PAGE_APP_SECRET=$(cat $SECRETS_PATH/single-page-app-auth0-secret.key)
AUTH0_NON_INTERACTIVE_CLIENT_ID=$(cat $SECRETS_PATH/non-interactive-auth0-client-id.key)
AUTH0_NON_INTERACTIVE_SECRET=$(cat $SECRETS_PATH/non-interactive-auth0-secret.key)
kubectl create secret -n koli-system generic auth0 \
  --from-file=auth0.crt=$AUTH0_SINGLE_PAGE_APP_CRT_PATH \
  --from-literal=spa-client-id.key=$AUTH0_SINGLE_PAGE_APP_CLIENT_ID \
  --from-literal=spa-secret.key=$AUTH0_SINGLE_PAGE_APP_SECRET \
  --from-literal=ni-client-id.key=$AUTH0_NON_INTERACTIVE_CLIENT_ID \
  --from-literal=ni-secret.key=$AUTH0_NON_INTERACTIVE_SECRET

# TODO: is this ssl secrets required???! (*.kolihub.io)
KOLIHUB_WILDCARD_CRT_PATH=$SECRETS_PATH/wildcard-kolihub-io.crt
KOLIHUB_WILDCARD_KEY_PATH=$SECRETS_PATH/wildcard-kolihub-io.key
kubectl create secret -n koli-system generic kong-ssl \
  --from-file=kong-default.crt=$KOLIHUB_WILDCARD_CRT_PATH \
  --from-file=kong-default.key=$KOLIHUB_WILDCARD_KEY_PATH 

# Cronjob billing
PAYPAL_SECRET_KEY=
PAYPAL_CLIENT_ID=
kubectl create secret -n koli-system generic paypal \
    --from-literal=client-id=$PAYPAL_CLIENT_ID \
    --from-literal=secret.key=$PAYPAL_SECRET_KEY
```

## Install / Configure platform components

```bash
# Add postgres database (kong/broker)
kubectl apply -f ./manifests/platform/postgres.yml

kubectl apply -f ./manifests/platform/kong/kong-init-config.yml
kubectl apply -f ./manifests/platform/kong/kong-migrations.yml

# Install kong, mutator and ingress controller
kubectl apply -f ./manifests/platform/kong/kong.yaml
kubectl apply -f ./manifests/platform/kong/k8smutator.yaml
kubectl apply -f ./manifests/platform/kong/kong-ingress.yaml

# Exec into the postgres pod and execute
psql -U system postgres -c "CREATE DATABASE broker"
psql -U system postgres -c "CREATE USER pgbroker WITH PASSWORD '$POSTGRES_PASSWORD'"
psql -U system postgres -c "GRANT ALL PRIVILEGES ON DATABASE broker to pgbroker"

kubectl apply -f ./manifests/platform/broker.yaml
kubectl apply -f ./manifests/platform/koli-controller.yaml
kubectl apply -f ./manifests/platform/gitstep.yaml

# Custom Configuration
kubectl apply -f ./manifests/platform/plans.yml
kubectl apply -f ./manifests/platform/default-domain.yml

# Koli Portal
CLIENT_ID=$(kubectl get secret -o template --template='{{index .data "spa-client-id.key"}}' -n koli-system auth0 |base64 -D)
kubectl apply -f ./manifests/platform/koli-portal-deploy.yml
kubectl patch deploy koli-portal --type='json' \
  '{"spec":{"containers":[{"name":"koli-portal","env":[{"name": "SPA_AUTH0_CLIENT_ID", "value": "'$CLIENT_ID'"}]}]}}'

# Daily Tracking and Invoice System
kubectl apply -f ./manifests/k8s/cronjobs/
```

# Install / Configure monitoring

## Prometheus Operator and Kube Prometheus

- [Prometheus Operator](https://github.com/coreos/prometheus-operator)
- [Kube Prometheus](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus)
- [`kubeadm` with Operator](https://github.com/coreos/prometheus-operator/blob/aaacaf555e75a563d8a7ed2a2e9c5ff4c05042ed/contrib/kube-prometheus/docs/kube-prometheus-on-kubeadm.md)

```bash
helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
helm install coreos/prometheus-operator --name prom-operator --namespace monitoring
helm install coreos/kube-prometheus --name kube-prom --set rbacEnable=true --namespace monitoring

# Fixes scraping of cadvisor metrics
kubectl apply -f ./manifests/k8s/fix-kube-prometheus-kubelet-scraping.yml
# Fixes alerting indicating scheduler and kube-controller-manager down
kubectl patch svc kube-prom-exporter-kube-scheduler -n kube-system --type='json' \
  -p='[{"op": "remove", "path": "/spec/selector/k8s-app"}]'
kubectl patch svc kube-prom-exporter-kube-controller-manager -n kube-system --type='json' \
  -p='[{"op": "remove", "path": "/spec/selector/k8s-app"}]'

# Configure Alerts to Slack
kubectl apply -f ./manifests/prom/kube-prom-alertmanager.yml

## Configure Platform Components

kubectl apply -f manifests/prom/service-monitors
kubectl apply -f manifests/prom/exporter-platform.yml

# Heapster https://github.com/kubernetes/heapster
helm install stable/heapster --name mon --set rbac.create=true --namespace kube-system
```

# Known Issues / Limitations

- Kube Prometheus: alerting and scrapping bugs - doesn't work out of box with `kubeadm`
  - [Issue 633](https://github.com/coreos/prometheus-operator/issues/633)
- Grafana: Some metrics need to be customized to be show properly
- EFS need to be created before applying the required manifests
- Broker database must be created on new installations (EFS with empty data)
