---
duration: 2h
---

# Run pipeline on Kubeflow Notebooks

Kubeflow Notebooks provides a flexible environment for developing machine learning pipelines. This project explores the creation, compilation, and execution of pipelines within the notebook. It highlights the integration between interactive development and orchestration. This enables a quick transition from prototype to deployment.

## Objectives

- How to develop and test model using Kubeflow Pipelines project on Kubeflow Notebooks.

## Tasks

1. Pipeline definition (medium level)
2. Pipeline submission (easy level)

## Prerequisites

Install Minikube and Kubeflow by following the [03.end-to-end-ml](../03.end-to-end-ml/lab-kubeflow.md) lab.

## Part 1. Pipeline definition (medium level)

Develop and test each step of your pipeline.
- load data: 
- preprocess
- train model
- evaluate

> Note: use the wine quality dataset (http://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/winequality-red.csv)

Use the KFP SDK to define your pipeline.

# Part 2. Pipeline submission (easy level)

- Run your pipeline using `kfp cli`. Run the command below on your notebook. 

```bash
!kfp run create \
    -e "<experiment name>" \
    -r "<run name>" \
    -f <compiled_file>.yaml
```

Replace `<experiment name>`, `<run name>` with your own texts, and `<compiled_file>` with generated IR YAML file.

- To explore generated artifacts click on submenus below `Pipelines` menu on the left side panel.

- Try different parameters to the model metrics.

- Change model and track his parameters.

## Further reading

- [Run pipeline](https://www.kubeflow.org/docs/components/pipelines/user-guides/core-functions/run-a-pipeline/)
