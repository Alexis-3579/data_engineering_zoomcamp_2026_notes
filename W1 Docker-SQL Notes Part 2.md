# Using Jupyter Notebook
We will now create a Jupyter Notebook notebook.ipynb file which we will use to read a CSV file and export it to Postgres.


### Setting up Jupyter Notebook 
```
uv add --dev jupyter

uv run jupyter notebook
```
- Note: if the above didn't work, and is giving an error ModuleNotFoundError: *No module named 'yaml'* try
  - `uv add pyyaml` and
  - `uv run --with pyyaml jupyter notebook`



### Exploring the dataset in Jupyter Notebook 

```
import pandas as pd

# Read a sample of the data
prefix = 'https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/'
dtype = {
    "VendorID": "Int64",
    "passenger_count": "Int64",
    "trip_distance": "float64",
    "RatecodeID": "Int64",
    "store_and_fwd_flag": "string",
    "PULocationID": "Int64",
    "DOLocationID": "Int64",
    "payment_type": "Int64",
    "fare_amount": "float64",
    "extra": "float64",
    "mta_tax": "float64",
    "tip_amount": "float64",
    "tolls_amount": "float64",
    "improvement_surcharge": "float64",
    "total_amount": "float64",
    "congestion_surcharge": "float64"
}

parse_dates = [
    "tpep_pickup_datetime",
    "tpep_dropoff_datetime"
]

df = pd.read_csv(
    prefix + 'yellow_tripdata_2021-01.csv.gz',
    nrows=100,
    dtype=dtype,
    parse_dates=parse_dates
)

```

### Using SqlAlchemy and Psycopg2 
```
!uv add sqlalchemy psycopg2-binary
```
- Double check in pyproject.toml to see that they've been added

Create database connection and get DDL schema 
```
from sqlalchemy import create_engine
engine = create_engine('postgresql://root:root@localhost:5432/ny_taxi')

print(pd.io.sql.get_schema(df, name='yellow_taxi_data', con=engine))
```
- Should get the output:
```
CREATE TABLE yellow_taxi_data (
    "VendorID" BIGINT,
    tpep_pickup_datetime TIMESTAMP WITHOUT TIME ZONE,
    tpep_dropoff_datetime TIMESTAMP WITHOUT TIME ZONE,
    passenger_count BIGINT,
    trip_distance FLOAT(53),
    "RatecodeID" BIGINT,
    store_and_fwd_flag TEXT,
    "PULocationID" BIGINT,
    "DOLocationID" BIGINT,
    payment_type BIGINT,
    fare_amount FLOAT(53),
    extra FLOAT(53),
    mta_tax FLOAT(53),
    tip_amount FLOAT(53),
    tolls_amount FLOAT(53),
    improvement_surcharge FLOAT(53),
    total_amount FLOAT(53),
    congestion_surcharge FLOAT(53)
)
```
Finally, to create the table (here we only create an empty table, not adding any data just yet): 
```
df.head(n=0).to_sql(name='yellow_taxi_data', con=engine, if_exists='replace')
```

### Ingesting the data 
Given there's 1369765 rows of data, we don't want to ingest the data all at once. Instead, we'll ingest it using an iterator in chunks. 




### Turning the Jupyter Notebook Into a Script 
```
uv run jupyter nbconvert --to=script Notebook.ipynb
```

Cleaning up and optimizing, using click to configure the parameters, the complete ingestion script is:
```
#!/usr/bin/env python
# coding: utf-8


import pandas as pd
from tqdm.auto import tqdm 
from sqlalchemy import create_engine
import click


# Read a sample of the data
prefix = 'https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/'
dtype = {
    "VendorID": "Int64",
    "passenger_count": "Int64",
    "trip_distance": "float64",
    "RatecodeID": "Int64",
    "store_and_fwd_flag": "string",
    "PULocationID": "Int64",
    "DOLocationID": "Int64",
    "payment_type": "Int64",
    "fare_amount": "float64",
    "extra": "float64",
    "mta_tax": "float64",
    "tip_amount": "float64",
    "tolls_amount": "float64",
    "improvement_surcharge": "float64",
    "total_amount": "float64",
    "congestion_surcharge": "float64"
}

parse_dates = [
    "tpep_pickup_datetime",
    "tpep_dropoff_datetime"
]


# Data Ingestion in Chunks 

@click.command()
@click.option('--year', type=int, default=2021, help='Year for the taxi data')
@click.option('--month', type=int, default=1, help='Month for the taxi data')
@click.option('--pg-user', default='root', help='PostgreSQL user')
@click.option('--pg-pass', default='root', help='PostgreSQL password')
@click.option('--pg-host', default='localhost', help='PostgreSQL host')
@click.option('--pg-port', type=int, default=5432, help='PostgreSQL port')
@click.option('--pg-db', default='ny_taxi', help='PostgreSQL database name')
@click.option('--target-table', default='yellow_taxi_data', help='Target table name')
@click.option('--chunksize', type=int, default=100000, help='Chunk size for reading CSV')
def run(year, month, pg_user, pg_pass, pg_host, pg_port, pg_db, target_table, chunksize):

    # create engine for sqlalchemy 
    engine = create_engine(f'postgresql://{pg_user}:{pg_pass}@{pg_host}:{pg_port}/{pg_db}')


    # read the csv and define the iterator 
    df_iter = pd.read_csv(
        prefix + f'yellow_tripdata_{year}-{month:02d}.csv.gz',
        dtype=dtype,
        parse_dates=parse_dates,
        iterator=True,
        chunksize=chunksize
    )

    first = True
    for df_chunk in tqdm(df_iter):
        if first:
            df_chunk.head(0).to_sql(name=target_table,
                                    con=engine,
                                    if_exists='replace'
                                    )
            first = False 
        df_chunk.to_sql(name='yellow_taxi_data', con=engine, if_exists='append')
        print(len(df_chunk))


if __name__ == "__main__": 
    run()

```

Example of usage of this script:
```
uv run python ingest_data.py \
  --pg-user=root \
  --pg-pass=root \
  --pg-host=localhost \
  --pg-port=5432 \
  --pg-db=ny_taxi \
  --target-table=yellow_taxi_trips
```

To dockerize:
```
 docker build -t taxi_ingest:v001 .
```




<img width="1069" height="576" alt="image" src="https://github.com/user-attachments/assets/8e52e879-37c7-4e10-83d2-a60bf4529b9e" />




