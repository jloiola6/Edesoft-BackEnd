# Insert S3 csv file content into DynamoDB

The following code snippet we can use it inside AWS lambda for fetching csv file content on S3 and store those data/values into DynamoDB
```
import json
import boto3
import uuid

import re
from datetime import datetime

s3_cient = boto3.client('s3')
dynamo_db = boto3.resource('dynamodb')
table = dynamo_db.Table('dados-csv')  # DynamoDB table name

def lambda_handler(event, context):
    # TODO implement
    bucket_name = event["bucket_name"]
    s3_file_name = event["object_key"]
    resp = s3_cient.get_object(Bucket=bucket_name, Key=s3_file_name)
    
    data = resp['Body'].read().decode('utf-8')

    employees = data.split("\n")
    for emp in employees:
        emp = emp.split(",")
        id = str(uuid.uuid4())
        # Exception handling done for skipping errors. When we are dealing with CSV file content, there may be a chance to get last row as empty row. So I have used try-catch for avoiding the runtime issues.
        try:                
            table.put_item(
                Item = {
                    "id": id,
                    "cpf": re.sub(r'\D', '', emp[0]),
                    "cnpj":  re.sub(r'\D', '', emp[1]),
                    "data":  str(datetime.strptime(emp[2].strip(), '%d-%m-%Y').strftime('%Y-%m-%d'))
                })
        except:
            continue
    return {
        'statusCode': 200,
        'body': json.dumps('Operação realizada com sucesso!')
    }

```
