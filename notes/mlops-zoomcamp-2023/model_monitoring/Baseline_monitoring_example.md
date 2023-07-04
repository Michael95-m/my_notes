# Baseline model for batch monitoring example

In this example, [Newyork city taxi dataset](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) is used. Green taxi dataset of Janurary 2021 will be used as training and validation datasets.

## Step-1: Model Creation using training and validation data

- Import the necessary libraries like pandas, datetime etc.

```python
import requests
import datetime
import pandas as pd

from joblib import load, dump
from tqdm import tqdm

from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_absolute_error, mean_absolute_percentage_error
```

- Download the dataset using **requests** library.

```python
files = [('green_tripdata_2022-02.parquet', './data'), ('green_tripdata_2022-01.parquet', './data')]

print("Download files:")
for file, path in files:
    url=f"https://d37ci6vzurychx.cloudfront.net/trip-data/{file}"
    resp=requests.get(url, stream=True)
    save_path=f"{path}/{file}"
    with open(save_path, "wb") as handle:
        for data in tqdm(resp.iter_content(),
                        desc=f"{file}",
                        postfix=f"save to {save_path}",
                        total=int(resp.headers["Content-Length"])):
            handle.write(data)
```

- Importing janauray dataset, create the target variable named **duration_min** and  remove outlier from the data.

```python
jan_data = pd.read_parquet('data/green_tripdata_2022-01.parquet')
```

```python
# create target
jan_data["duration_min"] = jan_data.lpep_dropoff_datetime - jan_data.lpep_pickup_datetime
jan_data.duration_min = jan_data.duration_min.apply(lambda td : float(td.total_seconds())/60)
```

```python
# filter out outliers
jan_data = jan_data[(jan_data.duration_min >= 0) & (jan_data.duration_min <= 60)]
jan_data = jan_data[(jan_data.passenger_count > 0) & (jan_data.passenger_count <= 8)]
```

