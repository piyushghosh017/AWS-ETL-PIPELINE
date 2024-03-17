# AWS-ETL-PIPELINE

About The Project:-
I) FETCH DATA FROM API AND STORING DATA IN S3 BUCKET USING AWS LAMBDA.

II) READ DATA FROM S3 BUCKET USING AWS GLUE AND STORE FROM_DATE ,TO_DATE IN MONGO DB

III) READ DATA FROM DYNAMO DB USING AWS GLUE

IV) CHECK HOW MANY RECORD AVAILABLE FROM S3 BUCKET AVAILABLE IN DYNAMO DB.

V) STORE UPCOMING NEW RECORDS ONLY IN DYNAMO DB.

VI) ARCHIEVE OLD DATA OF S3

### AWS-ETL-PROJECT- ARCHITECTURE

<img width="552" alt="ETL_architecture" src="https://github.com/piyushghosh017/AWS-ETL-PIPELINE/assets/75368732/bf85eebd-eb8f-4043-b865-6157039ab208">

### Built With

This section should list any major frameworks/libraries used to bootstrap your project. Leave any add-ons/plugins for the acknowledgements section. Here are a few examples.

* AWS
* PYSPARK 
#  *LETS START BUILDING PROJECT*

## Input Configuration For Web Api

  WEB API ---->FROM_DATE TO TO_DATE WE HAVE TO PASS IN WEB API URL TO FETCH JSON DATTA----> S3 BUCKET

## FIRST CREATE A CONDA  ENVIRONMENT USING CONDA IN VISUAL STUDIO TERMINAL.

```create -p venv python==3.9.12 -y
```
![ETL_1](https://github.com/piyushghosh017/AWS-ETL-PIPELINE/assets/75368732/3dc594dd-48b0-4787-9156-2efc08a637c5)

## THEN ACTIVATE CONDA ENVIRONMENT
```
conda activate venv/
```
![ETL_2](https://github.com/piyushghosh017/AWS-ETL-PIPELINE/assets/75368732/17d8634d-0463-4b54-b161-0e592aaa84e6)

## THEN INSTALL reuiresment.txt
```pip install -r requirements.txt
```
![ETL_3](https://github.com/piyushghosh017/AWS-ETL-PIPELINE/assets/75368732/11910bcb-82f1-4a2f-926f-32b06ebc0e88)

## Create a lambda_function_code folder and inside that folder lambda_function.py
code of lambda_function.py
```
import json
import pymongo
import certifi
import logging
import os
import boto3
import datetime
import os
import requests

ca = certifi.where()
import os
DATABASE_NAME = os.getenv("DATABASE_NAME")
COLLECTION_NAME = os.getenv("COLLECTION_NAME")
MONGODB_URL = os.getenv("MONGODB_URL")
BUCKET_NAME=os.getenv("BUCKET_NAME")

DATA_SOURCE_URL = f"https://www.consumerfinance.gov/data-research/consumer-complaints/search/api/v1/" \
                  f"?date_received_max=<todate>&date_received_min=<fromdate>" \
                  f"&field=all&format=json"
client = pymongo.MongoClient(MONGODB_URL, tlsCAFile=ca)

def get_from_date_to_date():
    from_date = "2023-01-01"
    from_date = datetime.datetime.strptime(from_date, "%Y-%m-%d")

    if COLLECTION_NAME in client[DATABASE_NAME].list_collection_names():

        res = client[DATABASE_NAME][COLLECTION_NAME].find_one(sort=[("to_date", pymongo.DESCENDING)])
        if res is not None:
            from_date = res["to_date"]

    to_date = datetime.datetime.now() #current date

    response = {
        "form_date": from_date.strftime("%Y-%m-%d"),
        "to_date": to_date.strftime("%Y-%m-%d"),
        "from_date_obj": from_date,
        "to_date_obj": to_date
    }
    logging.info(f"From date and to date {response}")
    return response

def save_from_date_to_date(data, status=True):
    data.update({"status": status})
    logging.info(f"saving from data and to date {data}")
    client[DATABASE_NAME][COLLECTION_NAME].insert_one(data)

def lambda_handler(event, context):
    print(event,context)
    from_date, to_date, from_date_obj, to_date_obj = get_from_date_to_date().values()
    if to_date==from_date:
        return {
            'statusCode': 200,
            'body': json.dumps('Pipeline has already downloaded all data upto yesterday')
        }
    url = DATA_SOURCE_URL.replace("<todate>", to_date).replace("<fromdate>", from_date)
    data = requests.get(url, params={'User-agent': f'your bot '})

    finance_complaint_data = list(map(lambda x: x["_source"],
                                    filter(lambda x: "_source" in x.keys(),
                                            json.loads(data.content)))
                                )
    s3 = boto3.resource('s3')
    s3object = s3.Object(BUCKET_NAME, f"inbox/{from_date.replace('-','_')}_{to_date.replace('-','_')}_finance_complaint.json")
    s3object.put(
        Body=(bytes(json.dumps(finance_complaint_data).encode('UTF-8')))
    )

    save_from_date_to_date({"from_date": from_date_obj, "to_date": to_date_obj})
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }


```

## THEN CONVERT YOUR CODE TO ZIP FILE TO UPLOAD IN LAMBDA USING BELOW

```
pip install --platform manylinux2014_x86_64 --target=<folder_name> --implementation cp --python 3.9 --only-binary=:all: --upgrade <lib1> <lib2>
```
EXAMPLE
```
pip install --platform manylinux2014_x86_64 --target=lambda_function_code --implementation cp --python==3.9.12 --only-binary=:all: --upgrade pymongo[srv] boto3 requests
```
![ETL_4](https://github.com/piyushghosh017/AWS-ETL-PIPELINE/assets/75368732/f0c65dc1-e7f0-4abd-ac2f-8f760180ab5c)


## CREATE A MONGODB ATLAS CLUSTER AND CONNECT TO  MONGODB COMPASS USING URL.

