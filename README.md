# pac-demo

a badly written hello world!

## How to guide

- Setup kind & ingress route
```bash
curl -s -L "https://raw.githubusercontent.com/snowdrop/k8s-infra/main/kind/kind-reg-ingress.sh" | bash -s y latest 0
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
  - host: webhook.192.168.1.90.nip.io
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
- Creating the secret needed by the controller
```bash
GITHUBAPP_PRIVATE_KEY=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/private_key)
GITHUBAPP_ID=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/app_id | awk 'NR==1{print $1}')
GITHUBAPP_WEBHOOK_SECRET=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/webhook_secret | awk 'NR==1{print $1}')
GITHUB_TOKEN=$(PASSWORD_STORE_DIR=~/.password-store-work pass show github/apps/my-pipeline-as-code/github_token | awk 'NR==1{print $1}')

kubectl delete secret/pipelines-as-code-secret -n pipelines-as-code
kubectl -n pipelines-as-code create secret generic pipelines-as-code-secret \
        --from-literal github-private-key="$(echo $GITHUBAPP_PRIVATE_KEY)" \
        --from-literal github-application-id="$GITHUBAPP_ID" \
        --from-literal webhook.secret="$GITHUBAPP_WEBHOOK_SECRET" \
        --from-literal github.token=$GITHUB_TOKEN
```
- Create the demo namespace, secret and repository
```bash
kubectl delete ns pac-demo
kubectl create ns pac-demo
kubectl delete secret/ch007m-pac-demo -n pac-demo
#kubectl -n pac-demo create secret generic ch007m-pac-demo \
#        --from-literal webhook.secret="$GITHUBAPP_WEBHOOK_SECRET" \
#        --from-literal github.token=$GITHUB_TOKEN

cat <EOF | kubectl apply -f -
apiVersion: v1
data:
  provider.token: Z2hwXzRTeTZzOVNCNkg0bWhsTDV2WWM4alJ1ZXNHeGNpdTFiTVpBNw==
  webhook.secret: ODE5YzdkZjNiZTVjZDMxMmVjMDVjYTg0ODRmZWQzNDNiMzE2MGFjMA==
kind: Secret
metadata:
  name: ch007m-pac-demo
  namespace: pac-demo
type: Opaque
EOF

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
      key: provider.token
      name: ch007m-pac-demo
    webhook_secret:
      key: webhook.secret
      name: ch007m-pac-demo
  url: https://github.com/ch007m/pac-demo       
```
- Git clone the demo project
```bash
rm -rf pac-demo
git clone https://github.com/ch007m/pac-demo ; cd pac-demo
git branch -D tektonci
git checkout -b tektonci

rm -rf .tekton ; mkdir -p .tekton
cat <<'EOF' > .tekton/pipelinerun.yaml
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: pac-demo
  annotations:
    # The event we are targeting as seen from the webhook payload
    # this can be an array too, i.e: [pull_request, push]
    pipelinesascode.tekton.dev/on-event: "[pull_request, push]"

    # The branch or tag we are targeting (ie: main, refs/tags/*)
    pipelinesascode.tekton.dev/on-target-branch: "[main]"

    # Fetch the git-clone task from hub, we are able to reference later on it
    # with taskRef and it will automatically be embedded into our pipeline.
    pipelinesascode.tekton.dev/task: "git-clone"

    # Use golangci-lint from the hub to test our Golang project
    pipelinesascode.tekton.dev/task-1: "golangci-lint"

    # You can add more tasks by increasing the suffix number, you can specify
    # them as array to have multiple of them.
    # browse the tasks you want to include from hub on https://hub.tekton.dev/
    #
    # pipelinesascode.tekton.dev/task-2: "[curl, buildah]"

    # how many runs we want to keep attached to this event
    pipelinesascode.tekton.dev/max-keep-runs: "5"
spec:
  params:
    # The variable with brackets are special to Pipelines as Code
    # They will automatically be expanded with the events from Github.
    - name: repo_url
      value: "{{ repo_url }}"
    - name: revision
      value: "{{ revision }}"
  pipelineSpec:
    params:
      - name: repo_url
      - name: revision
    workspaces:
      - name: source
      - name: basic-auth
    tasks:
      - name: fetch-repository
        taskRef:
          name: git-clone
        workspaces:
          - name: output
            workspace: source
          - name: basic-auth
            workspace: basic-auth
        params:
          - name: url
            value: $(params.repo_url)
          - name: revision
            value: $(params.revision)
      - name: golangci-lint
        taskRef:
          name: golangci-lint
        runAfter:
          - fetch-repository
        params:
          - name: package
            value: .
        workspaces:
          - name: source
            workspace: source

  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    # This workspace will inject secret to help the git-clone task to be able to
    # checkout the private repositories
    - name: basic-auth
      secret:
        secretName: "{{ git_auth_secret }}"
EOF

git add .tekton
git commit -asm "My first PAC"
git push --set-upstream origin tektonci
```
