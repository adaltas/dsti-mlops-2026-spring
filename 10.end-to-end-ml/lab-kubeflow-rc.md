---
duration: 1h30
---

# End-to-end ML Platform

Deploying machine learning solutions at scale requires robust, automated infrastructure. This project introduces the setup of a comprehensive Kubernetes-based platform for orchestrating containerized applications. By installing Minikube and then Kubeflow, it enables you to build a local environment dedicated to ML workflows and gain an understanding of the fundamentals of an end-to-end platform.

## Objectives

- Discover Kubernetes platform.
- Install entire Kubeflow platform.

## Tasks

If Minikube is already installed on your host machine and you have basic knowledges on Kubernetes, skip ahead to chapter 3.

1. Install Minikube (easy level)
2. Deployement of a simple application (easy level)
3. Install Kubeflow platforms (easy level)

## Part 1. Install Minikube (easy level)

Minikube is a tool that allows you to run a complete Kubernetes cluster locally on a single machine, which is ideal for learning, testing, and development. [Kubernetes](https://kubernetes.io/docs/concepts/overview/) is an open-source platform that orchestrates containers (such as Docker) on a cluster of machines to automate the deployment, scaling, and management of applications. In this lab, we will use Minikube to deploy a local Kubernetes environment and run machine learning workflows with Kubeflow. [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) is installed by following the instructions depending on your OS.

Start Minikube with:

```bash
minikube start \
  --cpus=6 \
  --memory=12288 \
  --disk-size=50g
```

Verify global status of cluster.

```bash
minikube status
kubectl get nodes
kubectl describe node <node-name>
```

Activate metrics server and show the resources consumption.

```bash
minikube addons enable metrics-server
```

Observe and explain what the command below does.

```bash
kubectl top pods -A --sort-by cpu --sum=true
```

Show resources deployed.

```bash
kubectl get pods -A
kubectl get deployments -A
kubectl get services -A
kubectl get namespaces
```

Explore `kube-system` namespace and try to understand the role of each component by reading the Kubernetes components overview [here](https://kubernetes.io/docs/concepts/overview/components/).

## Part 2. Deployement of a simple application (easy level)

Run the command below to create an application.

```bash
kubectl create deployment nginx --image=nginx
```

> NOTE: You may need to install `kubectl` following the instructions [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/). This should not be strictly necessary and you may substitute `kubectl` with `minikube kubectl` if preferred.

To allow external traffic with the deployment, it should be expose with:

```bash
kubectl expose deployment nginx --type=NodePort --port=80
```

Use the command below to access the application, then open web browser to reach it.

```bash
minikube service nginx
```

- What is the difference between a Pod and a Deployment?
- What is the role of a Service?

List resources.

```bash
kubectl get pods
kubectl get services
kubectl get deployments
```

- How many pods are created? 
- Why? ***Hint:*** Look into `Deployment` resource.

## Part 3. Install Kubeflow platforms (easy level)

Kubeflow platform is deployed by following the instructions below.

```bash
git clone https://github.com/kubeflow/manifests.git
cd manifests
git checkout tags/v1.11.0
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do echo "Retrying to apply resources"; sleep 20; done
```

Wait until all pods are in Running status.

```bash
kubectl wait pod \
--all \
--for=condition=Ready \
--namespace=kubeflow
kubectl get pods -n kubeflow
```

Running port forward allows to open the Kubeflow dashboard from web browser on [http://localhost:8080/](http://localhost:8080/) url.

```bash
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

Additionnal resources need to be added to enable the pipeline to run from the notebooks.

```bash
cat <<YAML | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubeflow-pipelines-runner
rules:
  - apiGroups: ["kubeflow.org"]
    resources:
      - experiments
      - runs
      - pipelines
    verbs:
      - get
      - list
      - create

  - apiGroups: ["argoproj.io"]
    resources:
      - workflows
    verbs:
      - get
      - list
      - watch

  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubeflow-pipelines-runner-binding
subjects:
  - kind: ServiceAccount
    name: default-editor
    namespace: kubeflow-user-example-com
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubeflow-pipelines-runner
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: additionnal-authorization-to-ml-pipeline
  namespace: kubeflow
spec:
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/kubeflow-user-example-com/sa/default-editor
    - when:
        - key: request.headers[kubeflow-userid]
          notValues:
            - "*"
  selector:
    matchLabels:
      app: ml-pipeline
---
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: add-header
  namespace: kubeflow-user-example-com
spec:
  configPatches:
    - applyTo: VIRTUAL_HOST
      match:
        context: SIDECAR_OUTBOUND
        routeConfiguration:
          vhost:
            name: ml-pipeline.kubeflow.svc.cluster.local:8888
            route:
              name: default
      patch:
        operation: MERGE
        value:
          request_headers_to_add:
            - append: true
              header:
                key: kubeflow-userid
                value: "user@example.com"
  workloadSelector: {}
YAML
```
