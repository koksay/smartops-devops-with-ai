# SmartOps: DevOps with AI Insights

## Minikube Deploy

Let's run the Ollama Models locally running Kubernetes Cluster:

```bash
# Clone this repo first
git clone git@github.com:koksay/smartops-devops-with-ai.git
cd smartops-devops-with-ai

minikube start --cpus=4 --memory=8192 --addons=ingress
minikube mount $PWD/models:/models
```

## Ollama on Kubernetes

Deploy Ollama on Minikube cluster:

```bash
# deploy ollama
kubectl apply -f ollama-deploy.yaml

# wait for pod to be ready
kubectl wait pod -l name=ollama -n ollama --for=condition=ready --timeout=600s

# port forward if not working with Ingress
kubectl port-forward svc/ollama 11434 -n ollama 

# ollama pull models
ollama pull llama2
ollama pull llama3

# list models available
ollama list
```

## K8sgpt CLI

Install k8sgpt via brew, or via any other installation method.

```bash
# list k8sgpt backends
k8sgpt auth list 

# add local ollama backend
k8sgpt auth add --backend localai --model llama2 --baseurl http://localhost:11434/v1

# make it default
k8sgpt auth default -p localai

# Create a problem
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
  namespace: default
spec:
  containers:
  - name: broken-pod
    image: nginx:1.a.b.c
    livenessProbe:
      httpGet:
        path: /
        port: 81
      initialDelaySeconds: 3
      periodSeconds: 3
EOF

# check issues
k8sgpt analyze --backend localai
## AI Provider: AI not used; --explain not set
##
## 0: Pod default/broken-pod()
## - Error: Back-off pulling image "nginx:1.a.b.c"

# explain it
k8sgpt analyze --backend localai --explain --no-cache # --> This took 35 sec
## AI Provider: localai
##
## 0: Pod default/broken-pod()
## - Error: Back-off pulling image "nginx:1.a.b.c"
##
## Error: Back-off pulling image "nginx:1.a.b.c" due to server unavailable. Retrying in 30s...
##
## Solution:
##
## 1. Check if the image exists in your Kubernetes cluster by running `kubectl get imagenginx:1.a.b.c`
## 2. If the image doesn't exist, try to create it using `kubectl create image nginx:1.a.b.c`
## 3. If the image exists but is unavailable, try to update the image tag by running `kubectl tag imagenginx:1.a.b.c nginx:1.a.b.c`
## 4. If the above steps fail, check the Kubernetes cluster logs for any errors or issues.

# update the backend with llama3 model
k8sgpt auth update --backend localai --model llama3 --baseurl http://localhost:11434/v1

# try llama3
k8sgpt analyze --backend localai --explain --no-cache # --> took 27 secs with llama3
## AI Provider: localai
##
## 0: Pod default/broken-pod()
## - Error: Back-off pulling image "nginx:1.a.b.c"
## Error: Back-off pulling image "nginx:1.a.b.c" ---.
##
## Solution: 
## 1. Check if the IP address "a.b.c" is correct.
## 2. If it's a private IP, ensure the Kubernetes cluster can access it.
## 3. Try to pull the image using `docker pull nginx:1.a.b.c` command.
## 4. If still failing, verify the DNS resolution for the IP address.
##
## Note: The error message suggests that the Kubernetes pod is trying to pull an image with an IP address as its tag, which is not a valid Docker image reference.

# Create more problems:
kubectl apply -f broken-deploy.yaml
```

## K8sgpt In-Cluster Operator

Deploy the operator:

```bash
helm upgrade --install k8sgpt-operator k8sgpt-operator \
  --repo https://charts.k8sgpt.ai/ \
  --namespace k8sgpt-operator-system \
  --create-namespace
```

Deploy k8sgpt resource:

```bash
kubectl apply -f - << EOF
apiVersion: core.k8sgpt.ai/v1alpha1
kind: K8sGPT
metadata:
  name: k8sgpt-sample
  namespace: k8sgpt-operator-system
spec:
  ai:
    enabled: true
    model: llama3
    backend: localai
    baseUrl: http://ollama.ollama.svc.cluster.local:11434
  noCache: false
  version: v0.3.41
  sink:
    type: slack
    webhook: ${SLACK_WEBHOOK_URL}
EOF
```

## Vulnerability Scanning

You can integrate `trivy`:

```bash
k8sgpt integration activate trivy
## ...
## 2024/10/29 08:51:52 release installed successfully: trivy-operator-k8sgpt/trivy-operator-0.24.1
## Activated integration trivy

k8sgpt filters list

# Check the report
k8sgpt analyze --filter VulnerabilityReport
## AI Provider: AI not used; --explain not set
##
## No problems detected

# Deploy a vulnerable application
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-pod
  namespace: default
spec:
  containers:
  - name: vulnerable-pod
    image: nginx:1.19.2
EOF

# Try again
k8sgpt analyze --filter VulnerabilityReport
```

## ShellGPT

```bash
sed -i 's/\(DEFAULT_MODEL=\).*/\1mistral:7b-instruct/' ~/.config/shell_gpt/.sgptrc
sed -i 's/\(API_BASE_URL=\).*/\1http:\/\/localhost:11434\/v1/' ~/.config/shell_gpt/.sgptrc


git diff | sgpt "Generate git commit message, for my changes"

kubectl describe pod broken-pod | sgpt "check logs, find errors, provide possible solutions"
## 1 Check the log for error messages using kubectl logs broken-pod -n default. Look for any issues related to image pullbackoff.
## 2 If the issue persists, ensure that the Minikube cluster has access to the Docker registry where the nginx:1.a.b.c image is located. You can do this by checking if the Docker daemon is running on your host machine and if it's connected to the Minikube cluster.
## 3 If the issue is with the network, try to connect to the registry manually from within the Minikube environment using docker pull nginx:1.a.b.c. If this fails, there might be a network configuration problem in your Minikube setup.
## 4 As a workaround, you can use a different image that is already available on the Minikube cluster or manually push the required image to the Minikube Docker daemon.
## 5 If none of the above solutions work, consider checking for any firewall rules or other system configurations that might be blocking the image pull.

sgpt --shell "find all yaml files in current folder"
##  find . -type f -name "*.yaml"
## [E]xecute, [D]escribe, [A]bort: E
## ./ollama-deploy.yaml

sgpt -s "start nginx container, mount ./index.html"
```
