---
duration: 2h
---

# Deploy a model on the Kubernetes platform

Deploying machine learning models at scale requires robust orchestration tools. This project leverages Kubernetes and Kubeflow KServe to expose a model as an inference service. These components illustrate a modern approach to ML deployment and provide insight into serving mechanisms in a distributed environment.

## Objectives

- Understand how to orchestrate the required Kubernetes components to deploy a model
- Learn to make a prediction service accessible from outside the cluster

## Task

1. Generate a model (medium level)
2. Create an InferenceService (easy level)

## Prerequisites

Install Minikube and Kubeflow by following the [03.end-to-end-ml](../03.end-to-end-ml/lab-kubeflow.md) lab.

## Part 1. Generate a model (medium level)

1. On your host machine, create and test your own python pipeline file named `train.py`. It will export the `.joblib` trained model and the `metrics.txt` files. 

2. After tests validation, change the path to the directory where generated files are stored. On the Kubernetes cluster, it will be `/mnt/model/`.

> Note: if using VM, copy your `train.py` file into th VM. Replace the variables `<USERNAME>`, `<LOCAL_HOST_PATH>` and  `<VM_IP>` with the right values. 

```bash
scp <LOCAL_HOST_PATH>/train.py <USERNAME>@<VM_IP>:/home/ubuntu/
```

Then, open another terminal to connect to VM using `ssh` command below. 

```bash
ssh <USERNAME>@<VM_IP>
```

3. Create Kubernetes resources

```bash
kubectl create configmap train-script \
--from-file=train.py \
-n kubeflow-user-example-com
cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-pvc
  namespace: kubeflow-user-example-com
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: batch/v1
kind: Job
metadata:
  name: wine-train
  namespace: kubeflow-user-example-com
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: trainer
        image: python:3.10
        command:
        - bash
        - -c
        - |
          pip install pandas scikit-learn && \
          python /workspace/train.py
        volumeMounts:
        - name: model-volume
          mountPath: /mnt/model
        - name: code-volume
          mountPath: /workspace
      volumes:
      - name: model-volume
        persistentVolumeClaim:
          claimName: model-pvc
      - name: code-volume
        configMap:
          name: train-script
YAML
#> configmap/train-script created
#> persistentvolumeclaim/model-pvc created
#> job.batch/wine-train created
```

Wait until the task is complete and check created resources.

```bash
kubectl wait --for=condition=Complete jobs wine-train -n kubeflow-user-example-com --timeout=300s
kubectl get jobs -n kubeflow-user-example-com
kubectl get pods -n kubeflow-user-example-com
```

3. Verify that the model and `metrics.txt` files have been created in `/mnt/model` folder. To do this, create a temporary pod that will mount the PVC.

```bash
cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-debugger
  namespace: kubeflow-user-example-com
spec:
  containers:
  - name: debugger
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: /model
      name: model-volume
  volumes:
  - name: model-volume
    persistentVolumeClaim:
      claimName: model-pvc
YAML
# pod/pvc-debugger created
kubectl exec -it pvc-debugger -n kubeflow-user-example-com -- sh -c "
  ls -la /model
  cat /model/*.txt 
"
```

4. Now, try to understand interactions between the different created objects.

5. Remove the temporary pod.

```bash
kubectl delete pod pvc-debugger -n kubeflow-user-example-com
```

## Part 2. Create an InferenceService (easy level)

1. An InferenceService is created to deploy the model created previously and stored in the volume mounted with PVC.

```bash
cat <<YAML | kubectl apply -f -
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: wine-elasticnet
  namespace: kubeflow-user-example-com
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: pvc://model-pvc
YAML
# inferenceservice.serving.kserve.io/wine-elasticnet created
kubectl wait --for=condition=Ready inferenceservice.serving.kserve.io wine-elasticnet -n kubeflow-user-example-com --timeout=300s
```

2. Check the status of InferenceService.

```bash
kubectl -n kubeflow-user-example-com get inferenceservices.serving.kserve.io
# NAME              URL                                                            READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION               AGE
# wine-elasticnet   http://wine-elasticnet.kubeflow-user-example-com.example.com   True           100                              wine-elasticnet-predictor-00001   28s
```

3. To be able to predict the wine quality, allow an external traffic with the predictor pod. To do that, a custom `NodePort` service will be created to expose the predictor pod to external client.

```bash
cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: wine-elasticnet-external-traffic
  namespace: kubeflow-user-example-com
spec:
  type: NodePort
  selector:
    serving.kserve.io/inferenceservice: wine-elasticnet
  ports:
  - name: http
    port: 80
    targetPort: 8012 
    protocol: TCP
YAML
```

4. Create a temporary network tunnel between your host machine and the pod predictor througth `NodePort` service. 

```bash
kubectl -n kubeflow-user-example-com port-forward svc/wine-elasticnet-external-traffic 8081:80
```

5. Open new terminal window and predict the targets of new inputs. It should return a list with predictions. 

> Note: if using VM, open another terminal and connect to VM using `ssh` command below. Replace the variables `<USERNAME>` and  `<VM_IP>` with the right values. 

```bash
ssh <USERNAME>@<VM_IP>
```

Use the model to make predictions based on a new dataset.

```bash
curl -v "http://localhost:8081/v1/models/wine-elasticnet:predict" \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [
      [1.2, 0.1, 0.5, 3.0, 2.0, 10.0, 33.0, 0.99, 3.0, 0.3, 9.3],
      [2.3, 0.8, 1.7, 13.0, 15.0, 72.7, 135.0, 1.007, 9.0, 1.8, 13.4],
      [1.3, 0.7, 1.1, 7.6, 13.4, 75.0, 67.0, 1.0, 6.7, 0.9, 12.2]
    ]
  }'
# * Host localhost:8081 was resolved.
# * IPv6: ::1
# * IPv4: 127.0.0.1
# *   Trying [::1]:8081...
# * Established connection to localhost (::1 port 8081) from ::1 port 38528 
# * using HTTP/1.x
# > POST /v1/models/wine-elasticnet:predict HTTP/1.1
# > Host: localhost:8081
# > User-Agent: curl/8.18.0
# > Accept: */*
# > Content-Type: application/json
# > Content-Length: 215
# > 
# * upload completely sent off: 215 bytes
# < HTTP/1.1 200 OK
# < date: Tue, 10 Mar 2026 20:07:03 GMT
# < server: uvicorn
# < content-length: 71
# < content-type: application/json
# < 
# * Connection #0 to host localhost:8081 left intact
# {"predictions":[5.349758856525737,5.958952824698714,6.073557268066545]}%   
```

## Further reading

- [Read more about Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/).
- [Persistant volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
- [Learn how Kubeflow KServe works](https://www.kubeflow.org/docs/components/kserve/introduction/).
