### Introduction
As we were able to have a stream of data through Kinesis Data service from Kinesis Video Stream, now it's time to store this data in DynamoDB database.

In order to store the data in the data base, we have few tasks before. In the following guide we will go through the necessary steps.

### DynamoDB: The Database

**Create the database**

To create the database, go to DynamoDb service, click on _**Create table**_. Then insert the table name (example: dataSteramTable). In the _**Partition Key**_ or the _**Primary Key**_ field, insert a key that it's value should be unique in your table(like an uuid). You can add a _**Sort Key**_ but it's optional, then click _**Create**_.

Now our table is ready. So let's create our lambda function.

### Lambda Function
The lambda function will be the interceptor or the connection between the data stream and the database.
When creating the lambda function, make sure to give the correct IAM role to be able to connect to DynamoDB and to get the data stream. 
Choose your runtime environment. Here we chose _**Python 3.6**_ then click _**create**_. It wil take few seconds. Now let's dive in the code.

> Code needs improvement.. NOT FINISHED but OPERATIONAL

1.  First let's import some libraries

```sh
import boto3
import datetime
import base64
import json
from boto3.dynamodb.conditions import Key, Attr
```
2. Now, let's create the handler function

We will comment the code below and explain every line
```sh
    now = str(datetime.datetime.now())
    print(now[0: 10])
    
    try:
        # connect to dynamoDb
        dynamo_db = boto3.resource('dynamodb')
        table = dynamo_db.Table('searchFaceResponse')

        # parse base64 data
        for record in event["Records"]:
            encoded = record["kinesis"]["data"]
            decoded = json.loads(base64.b64decode(encoded).decode("utf-8"))
            for faceSearchResponse in decoded["FaceSearchResponse"]:
                for matchedFaces in faceSearchResponse["MatchedFaces"]:
                    # parse data to dynamoDb item
                    data = {
                        'id' : str(now),
                        'FaceId': matchedFaces["Face"]["FaceId"],
                        'Confidence': str(matchedFaces["Face"]["Confidence"]),
                        'Similarity': str(matchedFaces["Similarity"]),
                        'ExternalImageId': matchedFaces["Face"]["ExternalImageId"],
                        'ImageId':matchedFaces["Face"]["ImageId"]
                    }
                    
                    # Scanning table, do a filter and get ExternalImageId
                    imageId = ''
                    res = table.scan(FilterExpression=Attr('ExternalImageId').eq('osm'))
                    data_res = res['Items']
                    for item in data_res:
                        imageId = item['ExternalImageId']
                        print(imageId)
                    #checking if the ExternalImageId already exists in the 
                    if data['ExternalImageId'] != imageId:
                        # send data to dynamoDb
                        table.put_item(Item=data)
    #handle exception
    except Exception as e: 
        print(str(e))
```
> Make sure the indentation is correct otherwise, you will get always errors. Yeah, it's Python !

Now, Start your Video Stream via GSTREAMER, and your stream processor. Wait a few seconds then go to your database, hit refresh button and see if you are getting the data.

> Another way to test you data stream and the data entry in DynamoDB, try to use Kinesis Data Generator(_Link below_). Click on help. Then click on _**Create a Cognito User with CloudFormation**_ button. It will create a CloudFormation stack. When deployed go to outputs and open the generated url to get access to the Generator.




#### Sources
- https://awslabs.github.io/amazon-kinesis-data-generator/web/producer.html




