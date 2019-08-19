# Objective
Build a cluster with redis (a key-value NoSQL data store) and mids base container, then use a python redis client library to connect to the redis server.

#### The yml file (docker-compose.yml)
This docker cluster contains Redis and Mids, with Mids supporting Jupyter Notebook on port 8888, with automated notebook startup.
```
---
version: '2'
services:
  redis:
    image: redis:latest
    expose:
      - "6379"

mids:
  image: midsw205/base:latest
  stdin_open: true
  tty: true
  volumes:
    - ~/w205:/w205
  expose:
    - "8888"
  ports:
    - "8888:8888"
  command: jupyter notebook --no-browser --port 8888 --ip 0.0.0.0 --allow-root
```

#### Download test dataset (bikeshare)
```
curl -L -o trips.csv https://goo.gl/QvHLKe
```

#### spin up the cluster
```
docker-compose up -d
```

#### Get token
```
docker-compose logs mids
```
Copy/paste jupyter notebook http string in browser, swap ip address

#### In notebook:
```python
import redis
import pandas as pd

trips = pd.read_csv('trips.csv')
date_sorted_trips = trips.sort_values(by='end_date')

for trip in data_sorted_trips.itertuples():
    print(trip.end_date, ' ', trip.bike_number, ' ', trip.end_station_name)

current_bike_locations = redis.Redis(host='redis', port='6379')

for trip in date_sorted_trips.itertuples():
    current_bike_locations.set(trip.bike_number, trip.end_station_name)

current_bike_locations.keys()
```

#### Find location of bike 92
```python
current_bike_locations.get('92')
```

#### Bring down cluster
```
docker-compose down
```
