## Unit Test

**Unit Testing** is checking a specific unit of the entire pipeline. 

To write unit test, our code shouldn't have much global variables (Instead of having global variables, we can create some class and function for that). And also large function should be divided into small functions.  Functions shouldn't be very long too. 

For unit test, **pytest** library is used. First, we need to create a folder named **tests** and add **__init__.py** which can be empty file inside that folder. 

We can install pytest by using pipenv like that.

```
pipenv install --dev pytest
```

We can write our test cases in files whose name starts with **test_** or ends with **_test.py**. pytest will look for these files in the specified directory.

Inside these files, we can write test functions. These functions' name must start with **test_** or ends with **_test**

Below is the sample test functions 

```python 
from pathlib import Path
import model

def read_text(file):
    test_directory = Path(__file__).parent

    with open(test_directory / file, 'rt', encoding='utf-8') as f_in:
        return f_in.read().strip()


def test_base64_decode():
    ## this test unit check the function of base64 decoding.
    base64_input = read_text('data.b64')

    actual_result = model.base64_decode(base64_input)
    expected_result = {
        "ride": {
            "PULocationID": 130,
            "DOLocationID": 205,
            "trip_distance": 3.66,
        },
        "ride_id": 256,
    }

    assert actual_result == expected_result

def test_prepare_features():
    ## this test unit check the function about preparing features
    model_service = model.ModelService(None)

    ride = {
        "PULocationID": 130,
        "DOLocationID": 205,
        "trip_distance": 3.66,
    }

    actual_features = model_service.prepare_features(ride)

    expected_fetures = {
        "PU_DO": "130_205",
        "trip_distance": 3.66,
    }

    assert actual_features == expected_fetures
```

In writing test units, we shouldn't depend on external data like model file.  So, in order to test the prediction function, we should create **ModelMock** class which predicts the imaginary prediction result.

```python
class ModelMock:
    def __init__(self, value):
        self.value = value

    def predict(self, X):
        n = len(X)
        return [self.value] * n


def test_predict():
    model_mock = ModelMock(10.0)
    model_service = model.ModelService(model_mock)

    features = {
        "PU_DO": "130_205",
        "trip_distance": 3.66,
    }

    actual_prediction = model_service.predict(features)
    expected_prediction = 10.0

    assert actual_prediction == expected_prediction

```

Last, we can run pytest like the following:

```bash
 ## tests is a folder where our test files exist
pytest tests/
```



