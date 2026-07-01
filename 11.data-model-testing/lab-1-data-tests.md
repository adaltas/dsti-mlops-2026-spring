---
duration: 2h
---

# Testing

Now we know how to version the data, the models, the environment, and how to link it to the source code. But we don't know if the data is of good quality and if the code behaves as it should. Let's write some tests!

## Objectives

- Learn how to define a test function and how to write an assertion.
- Learn the approach to data testing.
- Learn the approach to feature engineering testing.

## Tasks

1. Part 1. Dataset report with `data-profiling` (medium level)
2. Unit tests with `pytest` (medium level)

## Prerequisites

***If you are working locally on your host machine***

- Activate your virtual environment (`env_mlops`)
- Install: `pip install -U pytest`, `pip install fg-data-profiling`, `pip install ipywidgets`, `pip install ipytest` (the last one enables you to run pytest in a notebook), `pip install notebook`
- Start Jupyter Notebook: `jupyter notebook`. 
- From notebook upload the `testing_winequality.ipynb` file given in [lab-1-data-tests](./lab-1-data-tests) folder.

***If you are working on Kubeflow platform***

- Install Minikube and Kubeflow by following the [03.end-to-end-ml](../03.end-to-end-ml/lab-kubeflow.md) lab.
- On Kubeflow dashboard, create a new notebook, then upload the `testing_winequality.ipynb` file given in [lab-1-data-tests](./lab-1-data-tests) folder.
- On the top of your notebook, create a cell and install: `!pip install -U pytest`, `!pip install fg-data-profiling`, `!pip install ipywidgets`, `!pip install ipytest` (the last one enables you to run pytest in a notebook). 

## Part 1. Dataset report with `data-profiling` (medium level)

- Generate the reports about the datasets
- Explore them to see how they can help you getting the insights about the data
- How could you combine them with DVC and Git (data versioning)?

## Part 2. Unit tests with `pytest` (medium level)

- Test the functions (refresher)
- Test the datasets
- Test engineered features
