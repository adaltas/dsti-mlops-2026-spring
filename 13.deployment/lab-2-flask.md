---
duration: 2h
---

# Deploy a prediction model locally in a web application

This lab explores the creation of a local prediction service using Flask. It also highlights the importance of containerization with Docker, which enables the development of a portable and easily deployable application.

## Objectives

- Understand the steps of a model deployment using containerized apps 
- Learn how to integrate a model into web application
- Prepare and test the app locally
- Create a Docker container image

## Tasks

1. Prepare and test the app locally (easy level)
2. Create project package (medium level)
3. HTTP endpoint creation (easy level)
4. Create Docker image (easy level)

## Part 1. Prepare and test the app locally (easy level)

By now, we created and tested a model. But we only have a source code, which is not enough to deploy. First we need to save the model we want to deploy as a .bin file. Add the following code to the end of the `model.py` given ine the [lab-2-flask](./lab-2-flask) folder. Run it (in corresponding virtual enviromnent) to produce the pickled model.

``` python
import pickle
##dump the model into a file
with open("model.bin", 'wb') as f_out:
    pickle.dump(lr, f_out) # write final_model in .bin file
    f_out.close()  # close the file
```

## Part 2. Create project package (medium level)

1. Create your working directory `mkdir wine_app`.

2. Look into `model.py` and find out which libraries you are using. Find out the exact versions of these libraries and write them in the `requirements.txt`. Save it to the `wine_app` directory.

3. To your virt. env. install `flask` and `gunicorn`. Look up the exact versions and add them to the requirements.

4. Deactivate current virtual environment and create a new one: `conda create --name env_wine_app python=3.9` and activate it.
  
5. Install the requirements: `pip install -r requirements.txt`. That way you will try to create the minimal required environmenent (without libraries that you may installed, but are not needed for the project). 
 
6. Create `main.py` in `wine_app` directory:

```python
from flask import Flask

##creating a flask app and naming it "app"
app = Flask('app')
@app.route('/test', methods=['GET'])
def test():
    return 'Pinging Model Application!!'
if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=9696)
```

7. Run it in the terminal (be careful of folders and virt. env): `python main.py`.

8. Open the link written in the terminal and add `/test`. You will see if your setup is working or not.

9. Close it with `Ctrl+C`. If you close the terminal or open a new window, don't forget to reactivate the virtual environment: `conda activate env_wine_app`.  

10. In `wine_app` directory, create sub-directory `model_files`: `mkdir model_files`.

11. Inside `model_files` create `ml_model.py` and copy inside the `predict_wine_quality`function below. This is the prediction function that will be used to serve the model for predicting wine quality.

``` python
import pandas as pd

def predict_wine_quality(data, model):
    
    if type(data) == dict:
        df = pd.DataFrame(data)
    else:
        df = data
    #y_pred = model.predict(df)
    y_pred = model.predict(df).round().astype(int)
    return y_pred
```

12. Copy pickled model `model.bin` to `model_files`.

13. In `model_files`, create empty `__init__.py` file to indicate that the directory is a package.

## Part 3. HTTP endpoint creation (easy level)

Flask creates a REST API that exposes your trained model via HTTP endpoints (`/predict`) to receive data and return predictions.

1. Go back to the `main.py` and add the following code:

Imports, including your prediction function:

```python
import pickle
from flask import Flask, request, jsonify
from model_files.ml_model import predict_wine_quality
```

Copy `predict` function and it's route (before the line `if __name__ == '__main__'`) :

```python
@app.route('/predict', methods=['POST'])
def predict():
    wine_sample = request.get_json()
    print(wine_sample)
    with open('./model_files/model.bin', 'rb') as f_in:
        model = pickle.load(f_in)
        f_in.close()
    predictions = predict_wine_quality(wine_sample, model)
    result = list(predictions)
    # To be able to serialize it (int64 not serialisable)
    formatted_result = []
    for el in result:
        formatted_result.append(int(el))
    return jsonify(formatted_result)
```

2. Save and go to the terminal. Close the terminal window running from before (from the `main.py /test` test). If needed, reactivate the `env_wine_app` and go to the `wine_app` directory. There, run `main.py`.

3. Open new text file (doesn't matter where) and copy inside (`test_app_locally.py`): 

```python
import requests

new_data = {
    'fixed acidity': [1.2, 2.3, 1.3],
    'volatile acidity': [0.1, 0.8, 0.7],
    'citric acid': [0.5, 1.7, 1.1],
    'residual sugar': [3.0, 13.0, 7.6],
    'chlorides': [2.0, 15.0, 13.4],
    'free sulfur dioxide': [10.0, 72.7, 75.0],
    'total sulfur dioxide': [33.0, 135.0, 67.0],
    'density': [0.99, 1.007, 1.0],
    'pH': [3.0, 9.0, 6.7],
    'sulphates': [0.3, 1.8, 0.9],
    'alcohol': [9.3, 13.4, 12.2]
    }
url = "http://localhost:9696/predict"
r = requests.post(url, json = new_data)
r.text.strip()
print(r.text.strip())
```

When you run it, it should return a list with predictions. If you get an error saying thet the module `requests` was not found, `pip install requests` and run it again.

**NOTE:** If you didn't run the `main.py` before and left it running, this will not work!

4. Note that when you ran Flask, you got a following message: **WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.** This is due to the fact that the server, contained in the Flask, is not production-ready, but it's only meant for testing. For a production, we will use **unicorn server**.

5. (This step mignt not work on Windows, so you can skip it.) The command `gunicorn --bind=0.0.0.0:9696 main:app` will start the gunicorn server, as we started the Flask server before.    

## Part 4. Create Docker image (easy level)

1. Check out the examples of the Dockerfiles from `docker-containers`'s lab and create one to containerize your model. **Hint**: Instead of CMD, you will use `ENTRYPOINT ["gunicorn", "--bind=0.0.0.0:9696", "main:app"]` to start the app in the container.

2. Build the image and name it `wine-app`.

3. Run the container: `docker run -it --rm -p 9696:9696 wine-app:latest`

4. Test it again with the `test_app_locally.py` and stop it (`Ctrl+C`).

5. Tag the image properly and push it to the Docker Hub (check `docker-containers`'s lab).

## Further reading

- [Deploying Machine Learning Models with Flask and Docker](https://www.youtube.com/watch?v=KTd2a1QKlwo&list=WL&index=1)
