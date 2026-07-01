---
duration: 1h30
---

# Kubeflow Pipelines Standalone

## Install minikube

```bash
minikube start \
  --cpus=3 \
  --memory=6144 \
  --disk-size=50g
```

## Install Kubeflow Pipelines (KFP)

```bash
export PIPELINE_VERSION=2.16.1

kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/dev?ref=$PIPELINE_VERSION"
```

Wait all pods are ready.

```bash
kubectl wait pod \
--all \
--for=condition=Ready \
-n kubeflow
```

## Create tunnel to pipelines UI on another terminal

```bash
kubectl -n kubeflow port-forward svc/ml-pipeline-ui 8080:80
```

## Create kubernetes lab resources

```bash
K8S_LAB_POD="kfp-lab"
cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kfp-access: "true"
  name: kubeflow-pipelines-lab
---
apiVersion: v1
kind: Pod
metadata:
  name: kfp-lab
  namespace: kubeflow-pipelines-lab
spec:
  containers:
  - name: python
    image: python:3.10-slim
    command:
      - /bin/sh
      - -c
      - |
        pip install --upgrade pip && \
        pip install kfp && \
        sleep 3600
    env:
    - name: KF_PIPELINES_SA_TOKEN_PATH
      value: /var/run/secrets/kubeflow/pipelines/token
    volumeMounts:
    - name: workspace
      mountPath: /workspace
    - name: volume-kf-pipeline-token
      mountPath: /var/run/secrets/kubeflow/pipelines
      readOnly: true
  volumes:
  - name: workspace
    emptyDir: {}
  - name: volume-kf-pipeline-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 7200
          audience: pipelines.kubeflow.org
YAML
kubectl wait --for=condition=Ready pod $K8S_LAB_POD -n kubeflow-pipelines-lab --timeout=300s 
```

## Create pipeline

```bash
kubectl -n kubeflow-pipelines-lab exec $K8S_LAB_POD -- bash -c '
cat > /workspace/winequality_pipeline.py <<EOF
import kfp.compiler as compiler
from kfp import dsl
from kfp.dsl import Dataset, Input, Metrics, Model, Output, component, pipeline

@component(base_image="python:3.10", packages_to_install=["pandas", "scikit-learn"])
def get_dataset(
    train_data: Output[Dataset],
    test_data: Output[Dataset],
):
    import pandas as pd
    from sklearn.model_selection import train_test_split

    # Read the wine quality csv file from the URL
    csv_url =\
        "http://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/winequality-red.csv"
    df = pd.read_csv(csv_url, sep=";")
    # Split dataset into training and test sets. (0.75, 0.25) split.
    train, test = train_test_split(df)
    # Save the test and train dataset in separate files
    train.to_csv(train_data.path, index=False)
    test.to_csv(test_data.path, index=False)    
    print("Dataset loaded")

@component(
    base_image="python:3.10", packages_to_install=["pandas", "scikit-learn"]
)
def train(
    train_data: Input[Dataset],
    test_data: Input[Dataset],
    metrics: Output[Metrics],
    alpha: float = 0.5,
    l1_ratio: float = 0.5,
):
    import pandas as pd    
    import numpy as np
    from sklearn.linear_model import ElasticNet
    from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score
    import json
        
    train = pd.read_csv(train_data.path)
    train_x = train.drop(["quality"], axis=1)
    train_y = train[["quality"]]    
    test = pd.read_csv(test_data.path)
    test_x = test.drop(columns=["quality"])
    test_y = test["quality"]
    # Execute ElasticNet
    lr = ElasticNet(alpha=alpha, l1_ratio=l1_ratio, random_state=42)
    lr.fit(train_x, train_y)
    
    # Evaluate metrics
    test_y_pred = lr.predict(test_x)
    metrics_results = {
        "rmse": np.sqrt(mean_squared_error(test_y, test_y_pred)),
        "mae": mean_absolute_error(test_y, test_y_pred),
        "r2": r2_score(test_y, test_y_pred),
    }
    with open(metrics.path, "w") as f:
        json.dump(metrics_results, f, indent=4)
    print("Evaluation done:", metrics_results)

@pipeline(
    name="Wine-quality-pipeline",
    description="Elasticnet Pipeline with dependencies",
)
def wine_quality_pipeline(
    alpha: float = 0.5, 
    l1_ratio: float = 0.5
):
    collect_dataset_op = get_dataset()
    train_op = train(
        train_data=collect_dataset_op.outputs["train_data"],
        test_data=collect_dataset_op.outputs["test_data"],
        alpha=alpha,
        l1_ratio=l1_ratio
    )

if __name__ == "__main__":
  compiler.Compiler().compile(
    pipeline_func=wine_quality_pipeline, package_path="/workspace/winequality_pipeline.yaml"
  )
EOF
'
```

## IR YAML generation

```bash
kubectl -n kubeflow-pipelines-lab exec $K8S_LAB_POD -- bash -c '                                            
python3 /workspace/winequality_pipeline.py
'
```

## Run pipeline on pipeline UI

Upload IR YAML file in the pipeline UI after downloading it from the pod with the command below.

```bash
kubectl -n kubeflow-pipelines-lab cp "$K8S_LAB_POD":workspace/winequality_pipeline.yaml winequality_pipeline.yaml
```

## Run pipeline inside lab pod

```bash
kubectl -n kubeflow-pipelines-lab exec kfp-lab -- bash -c '
kfp run create \
  -e "Experiment of wine quality project - from kfp cli" \
  -r "Run of wine quality project - from kfp cli" \
  -f /workspace/winequality_pipeline.yaml
'
```

## Use host machine

Instead of using a working lab pod, you can create and run your logic on your host machine, generate IR YAML file by executing your python file locally, and then upload yaml file on the pipeline UI.
