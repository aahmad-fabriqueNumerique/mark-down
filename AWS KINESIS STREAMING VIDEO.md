# AWS VIDEO REKOGINITION VIDEO STREAM
After installing the _GSTREAMER, PRODUCER SDK_, we have to create a stream 

- Open [Amazon Kinesis Video Streams console](https://console.aws.amazon.com/kinesisvideo/home#/dashboard)
- Select a region where Amazon Kinesis Video Streams and Amazon Rekognition Video are available
- Click on the **_Video streams_** in the left menu and then click on the **_Create video stream_** button
- Fill in the form with the following information and click the **_Create video stream_** button

### Getting AWS Credentials
```sh
export AWS_DEFAULT_REGION="The region you use (e.g. ap-northeast-1, us-west-2)"
export AWS_ACCESS_KEY_ID="The AccessKeyId value"
export AWS_SECRET_ACCESS_KEY="The SecretAccessKey value"
```

### Download the video in MP4 format
- Open [Amazon Kinesis Video Streams console](https://console.aws.amazon.com/kinesisvideo/home?region=us-east-1#/)
- Click on **"streaming-name"** in the list of streams
- Click on the **_Media playback_** to expand it, then click on the **_Download clip_** in the bottom right

to use the AWS CLI, execute the following commands
```sh
aws kinesisvideo get-data-endpoint --stream-name your-stream-name --api-name GET_CLIP
```
You will get the following result
```sh
{
    "DataEndpoint": "https://b-xxxxxxxx.kinesisvideo.region.amazonaws.com"
}
```
Store this value in an environment variable.
```sh
export KVS_DATA_ENDPOINT="The DataEndpoint value above"
```
Then execute the following command.
```sh
# for Linux only
ONE_MIN_AGO=$(date -d "1 minute ago" -u "+%FT%T+0000")
NOW=$(date -u "+%FT%T+0000")

# for macOS only
ONE_MIN_AGO=$(date -v -1M -u "+%FT%T+0000")
NOW=$(date -u "+%FT%T+0000")

# in case you don't have the "date" command
ONE_MIN_AGO="2020-01-02T03:04:05+0900"
NOW="2020-01-02T03:05:05+0900"

# common to all OSes
aws kinesis-video-archived-media get-clip --endpoint-url $KVS_DATA_ENDPOINT \
--stream-name kvs-workshop-stream \
--clip-fragment-selector "FragmentSelectorType=SERVER_TIMESTAMP,TimestampRange={StartTimestamp=$ONE_MIN_AGO,EndTimestamp=$NOW}" \
kvs-workshop-cli.mp4
```
The command generates a file **_kvs-your-stream-cli.mp4_**

------

# ANALYZING VIDEO WITH AMAZON REKOGNITION VIDEO STREAM
### 1. Creating collection
- #### Preparing images for the collection
Start by taking some photos. The name of the image file should be something like **_name.jpg_** (e.g. alice.jpg). The name is used as the image ID when the image is added to the collection.

- #### Create S3 buckets and upload images for collection
    1- Open Amazon S3 console
    2- Select Buckets from the left menu and click the Create bucket button on the right
    3- Follow the default steps and and give a name for your bucket and check Block all public access
    4- Click on Create bucket
    5- Upload your photos into the bucket

- #### Creating a collection
Run the following commands on the terminal 
```sh
export COLLECTION_ID=kvs-name-collection
aws rekognition create-collection --collection-id $COLLECTION_ID
```
Confirm the following output and a collection named **_kvs-name-collection_** is created
```sh
{
    "StatusCode": 200,
    "CollectionArn": "aws:rekognition:your-region:012345678901:collection/kvs-name-collection",
    "FaceModelVersion": "5.0"
}
```

- #### Adding faces to the collection
Register the S3 bucket name you just created in an environment variable.
```sh
export BUCKET_NAME="kvs-your-bucket-name"
```
Run the following command to add all the images you just uploaded to the S3 bucket to the collection.
```sh
for key in $(aws s3 ls s3://$BUCKET_NAME/ | awk '{print $4}'); do
  name=$(echo $key | sed 's/\.[^\.]*$//')
  echo "index: $key"
  aws rekognition index-faces --collection-id $COLLECTION_ID \
  --image "S3Object={Bucket=$BUCKET_NAME,Name=$key}" \
  --external-image-id $name \
  --max-faces=1
done
````
Confirm the number of faces registered in the collection.
```sh
aws rekognition describe-collection --collection-id $COLLECTION_ID
```
You will get the following output, and check the value of FaceCount. If you can confirm the value of FaceCount as follows, which shows that there are two faces in the collection.
```sh
{
    "FaceCount": 26,
    "FaceModelVersion": "5.0",
    "CollectionARN": "arn:aws:rekognition:your-region:012345678901:collection/kvs-workshop-collection",
    "CreationTimestamp": 1588237022.79
}
```

### 2. Create a stream processor
create a stream processor, analyze video from Amazon Kinesis Video Streams with Amazon Rekognition Video, and handle the results using Amazon Kinesis Data Streams.

Now, Create a stream of Amazon Kinesis Data Streams to receive the results from the stream processor of Amazon Rekognition Video.
- #### Creating a stream in Amazon Kinesis Data Streams
    1- Open Amazon Kinesis Data Streams
    2- Select **_Data streams_** from the menu on the left and click on **_Create data stream_** on the right
    3- Enter a data stream name with a number of shards of 1.

Wait for the status of the stream to change from Creating to Active. 
After the stream is created, copy the ARN value of the stream’s details then run the following command to set the copied ARN in an environment variable.
```sh
export DATA_ARN="ARN of the data stream (e.g. arn:aws:kinesis:your-region:012345678901:stream/kvs-name-data-stream)"
```
- #### Creating an IAM Role
    1- Open IAM Roles Consle
    2- Select **_Roles_** from the menu on the left and click on **_Create role_** on the top
    3- Select type of trusted entity: **_AWS Services_** and choose a use case: **_Rekognition_** then click **_Permissions_**
    4- Click next **_Review_**
    5- Enter in **_Role name: kvs-name-rekognition-role_** then click **_create role_**
From the created role, copy the ARN and run the following command to to set the copied ARN in the environment variable.
```sh
export ROLE_ARN="IAM Role ARN (e.g.: arn:aws:iam::012345678901:role/kvs-name-rekognition-role)"
```
###### Attach a policy to the IAM role you have created:
- Click the Attach policies on the Permissions tab
- Search for kinesis in the policy filter and check the box to the left of AmazonKinesisFullAccess
- Click the Attach policy at the bottom of the screen


- #### Creating a stream processor

Once you have created a collection, a data stream, and an IAM role, you can create a stream processor for the collection.
First, copy the ARN for the Amazon Kinesis Video Streams stream.
- Open Amazon Kinesis Video Streams Console
- Click on the **_Video streams_** in the left menu > Click on **_kvs-name-stream_** in the list of streams
- Copy the value of the video stream ARN at the bottom of the page
- Execute the following command from the terminal and set the copied ARN to the environment variable.
```sh
export VIDEO_ARN="ARN of video stream (e.g. arn:aws:kinesisvideo:your-region:012345678901:stream/kvs-name-stream/1587989123456)"
```
Then create the stream processor by running the following command.
```sh
aws rekognition create-stream-processor \
  --input "KinesisVideoStream={Arn=$VIDEO_ARN}" \
  --name kvs-name-processor \
  --settings "FaceSearch={CollectionId=$COLLECTION_ID,FaceMatchThreshold=50.0}" \
  --role-arn $ROLE_ARN \
  --stream-processor-output "KinesisDataStream={Arn=$DATA_ARN}"
 ```
 If you get the result like below, a stream processor is created.
 ```sh
 {
    "StreamProcessorArn": "arn:aws:rekognition:your-region:012345678901:streamprocessor/kvs-workshop-processor"
}
```
Finally, run the following command to start the stream processor you created.
```sh
aws rekognition start-stream-processor --name kvs-name-processor
```

- #### Confirming the metrics
First, check that faces are recognized on Amazon Rekognition Video.
- Open Amazon Rekognition Console
- Click on the Metrics on the left menu
- Select the Last 4 hours at the top and make sure the call is successful and faces are detected, as shown below

Next, confirm that the recognition results have been sent to Amazon Kinesis Data Streams.
- Open Amazon Kinesis Data Streams Console
- Select **_Data streams_** from the menu on the left and click **_kvs-name-data-stream_**
- Open the Monitoring tab and make sure that data is being put.









