# pac-demo

a badly written hello world!

## How to guide

- Setup kind & ingress route
```bash
VM_IP=192.168.1.90
curl -s -L "https://raw.githubusercontent.com/snowdrop/k8s-infra/main/kind/kind-reg-ingress.sh" | bash -s y latest kind 0 ${VM_IP}

kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -f https://raw.githubusercontent.com/openshift-pipelines/pipelines-as-code/stable/release.k8s.yaml

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  labels:
    pipelines-as-code/route: controller
  name: pipelines-as-code
  namespace: pipelines-as-code
spec:
  ingressClassName: nginx
  rules:
  - host: webhook.${VM_IP}.nip.io
    http:
      paths:
      - backend:
          service:
            name: pipelines-as-code-controller
            port:
              number: 8080
        path: /
        pathType: Prefix
EOF
```

- Create a Github App
- Create a SMEE URL `https://smee.io/webhook.192.168.1.90.nip.io` `& Deployment
```bash
SMEE_URL=https://smee.io/webhook.${VM_IP}.nip.io
kubectl delete deployment/gosmee-client
cat <<EOF | kubectl apply -f -
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gosmee-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gosmee-client
  template:
    metadata:
      labels:
        app: gosmee-client
    spec:
      containers:
        - name: gosmee-client
          image: 'ghcr.io/chmouel/gosmee:main'
          args:
            - client
            - ${SMEE_URL}
            - http://pipelines-as-code-controller.pipelines-as-code.svc.cluster.local:8080
      restartPolicy: Always
EOF
```
- Creating the secret needed by the controller
```bash
GITHUBAPP_PRIVATE_KEY=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/private_key)
GITHUBAPP_ID=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/app_id | awk 'NR==1{print $1}')
GITHUBAPP_WEBHOOK_SECRET=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/webhook_secret | awk 'NR==1{print $1}')

kubectl delete secret/pipelines-as-code-secret -n pipelines-as-code
kubectl -n pipelines-as-code create secret generic pipelines-as-code-secret \
        --from-literal github-private-key="$(echo $GITHUBAPP_PRIVATE_KEY)" \
        --from-literal github-application-id="$GITHUBAPP_ID" \
        --from-literal webhook.secret="$GITHUBAPP_WEBHOOK_SECRET"
```
- Create the demo namespace, secret and repository
```bash
kubectl delete ns pac-demo
kubectl create ns pac-demo

GITHUB_TOKEN=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/github_token | awk 'NR==1{print $1}')
kubectl delete secret/ch007m-pac-demo -n pac-demo
kubectl -n pac-demo create secret generic ch007m-pac-demo \
        --from-literal webhook.secret=$GITHUBAPP_WEBHOOK_SECRET \
        --from-literal github.token=$GITHUB_TOKEN

kubectl delete repositories.pipelinesascode.tekton.dev/ch007m-pac-demo -n pac-demo
cat <<EOF | kubectl apply -f -
apiVersion: pipelinesascode.tekton.dev/v1alpha1
kind: Repository
metadata:
  name: ch007m-pac-demo
  namespace: pac-demo
spec:
  git_provider:
    secret:
      key: github.token
      name: ch007m-pac-demo
    webhook_secret:
      key: webhook.secret
      name: ch007m-pac-demo
  url: https://github.com/ch007m/pac-demo
EOF         
```
- Git clone the demo project
```bash
rm -rf pac-demo
git clone https://github.com/ch007m/pac-demo ; cd pac-demo
git checkout main
git push -d origin tektonci
git branch -d tektonci
git checkout -b tektonci

mkdir -p .tekton
wget https://raw.githubusercontent.com/ch007m/pac-demo/main/.tekton/pipelinerun.yaml -O k8s/pipelinerun.yaml

git add .tekton
git commit -asm "My first PAC"
git push --set-upstream origin tektonci
```
