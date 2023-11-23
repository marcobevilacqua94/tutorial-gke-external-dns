##### TUTORIAL GKE CONFIGURING EXTERNAL DNS

##### INSTALLING OF GCLOUD COMMAND LINE ON UBUNTU 20.4
```
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates gnupg curl sudo
echo "deb [signed-by=/usr/share/keyrings/cloud.google.asc] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo tee /usr/share/keyrings/cloud.google.asc
sudo apt-get update && sudo apt-get install google-cloud-cli

gcloud init
```
##### AND LOG IN GOOGLE CLOUD ACCOUNT

##### INSTALLING OF KUBECTL
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

sudo apt-get install kubectl
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```
##### SETTING THE ENVIRONMENT (REPLACE PROJECTID AND REGION AND CLUSTER-NAMESPACE VARIABLES)
```
gke-gcloud-auth-plugin --version

export GKE_PROJECT_ID="<projectid>"
export DNS_PROJECT_ID="<projectid>"
export DNS_SA_NAME="external-dns-sa"
export DNS_SA_EMAIL="$DNS_SA_NAME@${GKE_PROJECT_ID}.iam.gserviceaccount.com"

export GKE_CLUSTER_NAME="my-worker-node-cluster"
export GKE_SA_NAME=$GKE_CLUSTER_NAME-sa
export GKE_SA_EMAIL="$GKE_SA_NAME@${GKE_PROJECT_ID}.iam.gserviceaccount.com"
export GKE_REGION="<REGION>"

export KUBECONFIG=~/.kube/${GKE_CLUSTER_NAME}.${GKE_REGION}.yaml
export EXTERNALDNS_NS="kube-addons"
export NGINX_NS="my-nginx"
export CLUSTER_NS = "<cluster-namespace>"
```
##### THIS DOMAIN NAME MUST BE REGISTERED ON A REGISTRAR LIKE GODADDY OR NAMECHEAP OR CLOUDFLARE
##### THEN YOU HAVE TO CONFIGURE DNS WITH A PROVIDER SUPPORTED BY EXTERNAL DNS COMPONENT (LIKE CLOUDFLARE)
##### YOU CAN ALSO INSERT IN CLOUDFLARE THE DOMAIN REGISTERED WITH ANOTHER REGISTRAR AND THEN TELL THE REGISTRAR TO USE CLOUDFLARE SERVERS (LIKE HERE https://www.namecheap.com/support/knowledgebase/article.aspx/9607/2210/how-to-set-up-dns-records-for-your-domain-in-cloudflare-account/)

##### REPLACE DOMAIN VARIABLE
```
export DOMAIN_NAME="<DOMAIN>"
```
##### CREATE CLOUD DNS ZONE
```
gcloud dns managed-zones create ${DOMAIN_NAME/./-} --project $DNS_PROJECT_ID \
  --description $DOMAIN_NAME --dns-name=$DOMAIN_NAME --visibility=public
```
##### SAVE LIST OF NAME SERVERS
```
NS_LIST=$(gcloud dns record-sets list --project ${DNS_PROJECT_ID} \
  --zone "${DOMAIN_NAME/./-}" --name "$DOMAIN_NAME." --type NS \
  --format "value(rrdatas)" | tr ';' '\n')
```
##### PRINT LIST OF NAME SERVERS
```
echo "$NS_LIST"
```
##### PREPARE A SERVICE ACCOUNT WITH THE LEAST PRIVILEGES REQUIRED FOR WORKER NODE
```
ROLES=(
  roles/logging.logWriter
  roles/monitoring.metricWriter
  roles/monitoring.viewer
  roles/stackdriver.resourceMetadata.writer
)

gcloud iam service-accounts create $GKE_SA_NAME \
  --display-name "gke-worker-nodes" --project $GKE_PROJECT_ID

for ROLE in ${ROLES[*]}; do
  gcloud projects add-iam-policy-binding $GKE_PROJECT_ID \
    --member "serviceAccount:$GKE_SA_EMAIL" \
    --role $ROLE
done
```
##### CREATE THE CLUSTER WITH THE SERVICE ACCOUNT
```
gcloud container clusters create $GKE_CLUSTER_NAME \
  --project $GKE_PROJECT_ID --region $GKE_REGION --num-nodes 3 \
  --service-account "$GKE_SA_EMAIL"
```
##### CREATE NAMESPACES FOR DNS AND NGINX (NGINX USED TO TEST)
```
NAMESPACES=(
  $EXTERNALDNS_NS
  $NGINX_NS
)

for NAMESPACE in ${NAMESPACES[*]}; do
  kubectl get namespaces | grep -q $NAMESPACE || \
    kubectl create namespace $NAMESPACE
done
```
##### GIVE THE SERVICE ACCOUNT THE DNS ADMIN ROLE 
```
gcloud projects add-iam-policy-binding $DNS_PROJECT_ID \
  --member serviceAccount:$GKE_SA_EMAIL \
  --role roles/dns.admin
```
##### CREATE externaldns.yaml (REMOVE THE RESOURCES YOU DO NOT WANT FROM RULES) (THIS SETUP USES CLOUDFLARE) (REPLACE API-KEY AND API-EMAIL AND DOMAIN AND PROJECTID VARIABLES)
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods","nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  labels:
    app.kubernetes.io/name: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: kube-addons
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: external-dns
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-dns
    spec:
      serviceAccountName: external-dns
  #    nodeSelector:
  #      iam.gke.io/gke-metadata-server-enabled: "true"
      containers:
        - name: external-dns
          image: k8s.gcr.io/external-dns/external-dns:v0.11.0
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=<DOMAIN>
            - --provider=cloudflare
            - --google-project=<PROJECT-ID>
            - --google-zone-visibility=public
            - --policy=upsert-only
            - --registry=txt
            - --txt-owner-id=external-dns
          env:
            - name: CF_API_KEY
              value: <API-KEY>
            - name: CF_API_EMAIL
              value: <API-EMAIL>
```
##### GET THE POD NAMES TO SEE LOGS
```
POD_NAME=$(kubectl get pods \
  --selector "app.kubernetes.io/name=external-dns" \
  --namespace kube-addons --output name)
```
##### CHECK LOGS FOR ISSUES, IT SHOULD SAY ALL RECORDS ADDED 
```
kubectl logs $POD_NAME --namespace kube-addons
```
##### VERIFY CREATING A NGINX SERVER, CREATE nginx.yaml (REPLACE DOMAIN VARIABLE)
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.<DOMAIN>
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: http
```
##### CREATE AND CHECK DNS IS WORKING
```
kubectl create -f nginx.yaml --namespace ${NGINX_NS:-"default"}

kubectl get service --namespace ${NGINX_NS:-"default"}

gcloud --project $DNS_PROJECT_ID dns record-sets list \
  --zone "${DOMAIN_NAME/./-}" --name "nginx.${DOMAIN_NAME}."
```
##### YOU SHOULD SEE THE DNS RECORD 
```
NAME_SERVER=$(head -1 <<< $NS_LIST)
dig +short @$NAME_SERVER nginx.$DOMAIN_NAME
dig +short nginx.$DOMAIN_NAME
```
##### DOWNLOAD AND INSTALL COUCHBASE OPERATOR
```
wget https://packages.couchbase.com/couchbase-operator/2.5.0/couchbase-autonomous-operator_2.5.0-kubernetes-linux-amd64.tar.gz

tar -xvzf couchbase-autonomous-operator_2.5.0-kubernetes-linux-amd64.tar.gz
cd couchbase-autonomous-operator_2.5.0-180-kubernetes-linux-amd64/
kubectl apply -f crd.yaml
bin/cao create admission
bin/cao create operator
```
##### CREATE CERTIFICATES
```
git clone https://github.com/OpenVPN/easy-rsa

cd easyrsa3 

./easyrsa init-pki

./easyrsa build-ca

./easyrsa --subject-alt-name="DNS:*.${DOMAIN_NAME},DNS:*.${DOMAIN_NAME}.${CLUSTER_NS},DNS:*.${DOMAIN_NAME}.${CLUSTER_NS}.svc,DNS:*.${CLUSTER_NS}.svc,DNS:${DOMAIN_NAME}-srv,DNS:${DOMAIN_NAME}-srv.${CLUSTER_NS},DNS:${DOMAIN_NAME}-srv.${CLUSTER_NS}.svc,DNS:localhost,DNS:*.${CLUSTER_NS}.${DOMAIN_NAME},DNS:*.${CLUSTER_NS}.svc.cluster.local,DNS:*.${CLUSTER_NS}.svc.cluster,DNS:*.${CLUSTER_NS}.info,DNS:*.${CLUSTER_NS}.svc.info,DNS:*.cluster,DNS:*.cluster.${CLUSTER_NS},DNS:*.cluster.${CLUSTER_NS}.svc,DNS:*.cluster.${CLUSTER_NS}.svc.cluster.local,DNS:cluster-srv,DNS:cluster-srv.${CLUSTER_NS},DNS:cluster-srv.${CLUSTER_NS}.svc,DNS:*.cluster-srv.${CLUSTER_NS}.svc.cluster.local" build-server-full couchbase-server nopass

cd ../..

cp easy-rsa/easyrsa3/pki/private/couchbase-server.key pkey.key
 
cp easy-rsa/easyrsa3/pki/issued/couchbase-server.crt chain.pem
 
cp easy-rsa/easyrsa3/pki/ca.crt ca.crt

openssl rsa -in pkey.key -out pkey.key.der -outform DER

openssl rsa -in pkey.key.der -inform DER -out pkey.key -outform PEM
```
##### CREATE CLUSTER NAMESPACE
```
kubectl create namespace $CLUSTER_NS
```
##### LOAD CERTIFICATES
```
kubectl create secret generic couchbase-server-tls -n $CLUSTER_NS --from-file chain.pem --from-file pkey.key

kubectl create secret generic couchbase-operator-tls -n $CLUSTER_NS --from-file ca.crt
```
##### CREATE CLUSTER ADMISSION AND OPERATOR
```
bin/cao create admission --namespace $CLUSTER_NS

bin/cao create operator --namespace $CLUSTER_NS
```
##### CREATE couchbase.yaml FOR THE CLUSTER, REPLACE CLUSTER-NS AND DOMAIN VARIABLES
```
apiVersion: v1
kind: Secret
metadata:
  name: cb-example-auth
type: Opaque
data:
  username: QWRtaW5pc3RyYXRvcg== # Administrator
  password: cGFzc3dvcmQ=         # password
---
apiVersion: couchbase.com/v2
kind: CouchbaseBucket
metadata:
  name: default
---
apiVersion: couchbase.com/v2
kind: CouchbaseCluster
metadata:
  name: cluster
spec:
  image: couchbase/server:7.2.0
  security:
    adminSecret: cb-example-auth
  buckets:
    managed: true
  servers:
  - size: 3
    name: all_services
    services:
    - data
    - index
    - query
    - search
    - eventing
    - analytics
  networking:
    exposeAdminConsole: true
    adminConsoleServiceTemplate:
      spec:
        type: LoadBalancer
    exposedFeatures:
    - xdcr
    - client
    exposedFeatureServiceTemplate:
      spec:
        type: LoadBalancer
    dns:
      domain: <CLUSTER-NS>.<DOMAIN>
    tls:
      static:
        serverSecret: couchbase-server-tls
        operatorSecret: couchbase-operator-tls
```
```
kubectl create -f couchbase.yaml -n $CLUSTER_NS
```
##### CLUSTER SHOULD BE RECHABLE, CHECK NETWORKING IF NOT
```
./sdk-doctor-linux diagnose couchbases://console.${CLUSTER_NS}.${DOMAIN_NAME}/default -u Administrator -p password
```
