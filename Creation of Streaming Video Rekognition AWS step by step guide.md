# Facial Recognition with Kinesis Video Streams
After reading  several articles about how to create facial recognition system using Kinesis Video Stream, I present a guide to show the necessary steps to create this system (in spite of the poor results that I got during the process) of my point of view.
The steps contain bo errors but the results still uncertain until this moment (24/03/2021) due to the lack of of complete guide to building something like this anywhere online 😓. It is has been a real struggle but we are still trying.🤞 

![Tux, the Linux mascot](https://miro.medium.com/max/700/0*hmKKGDLoGvR6ATkc)

We will be trrying to make the same system as shown in the figure above... so let's start 💪
## Step. 1

- #### Configure the AWS CLI
```sh
# Install the AWS CLI
$ sudo pip install awscli
```
If you want to install the aws-sdk (cli) using other methods, please consider reading the documentaion of the installation in this link [Installing, updating, and uninstalling the AWS CLI version 2 on Linux](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html)

```sh
# Configure the CLI
$ aws configure
```
During the configure process you will need your “***Access Key ID***” and your “***Secret Access Key***”. You can also set your preferred AWS region (e.g. eu-west-1 for Ireland), and a preferred output format (“json”).
You can store your keys in an environment variable or in the bashrc file
```sh
export AWS_ACCESS_KEY=<YourAccessKeyId>
export AWS_SECRET_ACCESS_KEY=<YourSecretAccessKey>
export rAWS_REGION=<your-region>
```
### Step. 2
- ### Create a Kinesis Video Stream

In order to create the video stream, you'll need to set a name and data retention settings:
```sh
$ aws kinesisvideo create-stream --stream-name your-steram-name --data-retention-in-hours 1
```
**Data retention** means you can keep (and view) any data that in a Kinesis Video Stream for the chosen amount of time.  Predictably, however, there is a cost associated with this as long as the stream is active even though there is no live video streaming.

- ### Streaming to Kinesis

To create and send data to the stream, we will need the **Producer SDK - GSTREAMER** from AWS. I will refer to my step by step doc explaining how to install and configure the **Producer SDK - GSTREAMER**. You can find it by clicking here to acces the repository on github: [AWS KINESIS GSTREAMER SETUP](https://github.com/aahmad-fabriqueNumerique/mark-down/blob/main/AWS%20KINESIS%20GSTREAMER%20SETUP.md).

Then, you can start streaming using this command
```sh
$ gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! video/x-raw,format=I420,width=640,height=480 ! x264enc bframes=0 key-int-max=45 bitrate=512 tune=zerolatency ! h264parse ! video/x-h264,stream-format=avc,alignment=au,profile=baseline ! kvssink stream-name="MyKinesisVideoStream" storage-size=512 access-key="YourAccessKey" secret-key="YourSecretKey" aws-region="YourAWSRegion"
```

### Step. 3
- ### Face Detection with Amazon Rekognition

Before we can do any detections, we need to train Rekognition to recognise some faces. So, we need to create a new collection — somewhere we can store our trained faces:
```sh
$ aws rekognition create-collection --collection-id your-collection-name
```
A successful response will return you the ARN for this collection. You will need this later on but can easily be retrieved by using this command:
```sh
$ aws rekognition describe-collection --collection-id your-collection-name
```
To give it some images for training, we'll use this command: (make sure to have some images of some faces)
```sh
$ aws rekognition index-faces --collection-id your-collection-name --external-image-id YourImageFaceName --image-bytes fileb://yourImg.jpg
```
What does this command do? I'll explain:
- index-faces: we are telling AWS Rekognition to index some face the we will provide
- collection-id: We are specifying the name of th collection that we have just created
- external-image-id: this parameter will tell rekognition to whom belong this image. So we can refer multiple images to the same id
- image-bytes: refers to the path of the image. the fileb:// parameter specifies that the file is a binary file.

The response of this commands will conatain some data about any faces that are detected in the image, like the faceId, Confidence..etc. This hwo it should look like:
```sh
[
  {
    "BoundingBox": {
      "Width": 0.584558968574582, 
      "Top": 0.32326598542187, 
      "Left": 0.298685365974515, 
      "Height": 0.277484585459636
    }, 
    "FaceId": "jfuir54-ff5e-4548-r105-r135zfsg687e", 
    "ExternalImageId": "YourImageFaceName", 
    "Confidence": 99.9999, 
    "ImageId": "kfgzef5468-fse5-336e-eee8-esz45es548"
  }
]
```
The _**FaceId**_ relates to the Rekognition unique ID for the detected face, whereas the _**ImageId**_ relates to the image this face was detected in. The _**externalImageId**_ is the ID we supplied during the indexing process.

To find all related faces by using the FaceId, we use this command:
```sh
aws rekognition search-faces --collection-id your-collection-name --max-faces 1 --face-match-threshold 70 --face-id jfuir54-ff5e-4548-r105-r135zfsg687e
```
The response will be an array of indexed faces that are similar to the _**faceId**_ we provided before :
- face-match-threshold: is used to filter out poor matches
- max-faces: is to limit the number of results returned

To verify that AWS Rekognition has done the job,we run the following command:
```sh
aws rekognition search-faces-by-image --collection-id your-collection-name --image-bytes fileb://yourImg.jpg
```
This will return an array of face matches, starting with the best match.

- ### Connecting Kinesis Video Streams to Amazon Rekognition

In this step we will try to build a link between AWS Rekognition and Kinesis Video Stream. So, to do that, we need to create Rekognition Stream Processor.
The Stream Processor is part of Rekognition Video. It takes an input (the Kinesis Video Stream), and an output (a Kinesis Data Stream). 

- ##### Creating Kinesis Data Stream
So, first thing is to create a data stream
```sh
# Create the stream (no response given)
aws kinesis create-stream --stream-name your-data-stream-name --shard-count 1
```
to list the stream, we run this command:
```sh
# list the current streams to verify it has been created
aws kinesis list-streams
```
and the output should be like this:
```sh
{
    "StreamNames": [
        "your-date-stream-name"
    ]
}
```
To get more info about the stream... we can run the following command:
```sh
aws kinesis describe-stream --stream-name your-date-stream-name
```
Make sure to copy the ARN which will appear in the output because we'll need it later.

- ##### Create new IAM role

We need Rekognition to be able to read data from a Kinesis Video Stream, and write data to a Kinesis Data Stream. So, in the IAM page we have to create a new policy. Copy and paste the following JSON in the JSON editor
```sh
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "kinesis:PutRecord",
                "kinesis:PutRecords",
                "kinesisvideo:GetDataEndpoint",
                "kinesisvideo:GetMedia"
            ],
            "Resource": [
                "arn:aws:kinesisvideo:*:*:stream/*/*",
                "arn:aws:kinesis:*:*:stream/*"
            ]
        }
    ]
}
```
What this policy is doing?
* Allowing access to the PutRecord method for Amazon Kinesis
* Allowing access to the PutRecords method for Amazon Kinesis
* Allowing access to the GetDataEndpoint method for Amazon Kinesis Video
* Allowing access to the GetMedia method for Amazon Kinesis Video
* Enable these methods on any Kinesis or Kinesis Video stream

Then give your policy a name when asked "MyKinesisFaceRekPolicy"

Next, we need to create a new Role for an AWS Service. Select Rekognition, and move to the next screen. At this point you are forced to select the “AmazonRekognitionServiceRole” Policy. However this policy has limited access to resources whose name begins with “AmazonRekognition”. Additionally, we are not allowed to add our own policies.

Now, once the role is created, we will be able to to remove the unwanted “AmazonRekognitionServiceRole” Policy and attach our own custom Policy.

This Role allows Rekognitiono use other AWS Services inline with the Policy we created. In this instance, it allows access to some of the methods available for Kinesis Video and Data streams.

- #### Creating a Stream Processor

Here, we have to provide a lot of parameters and some values. To facilitate the work, we can send a JSON file conatains all these values.
So, we start by generating a template and fill it with our information. Run the following command:
```sh
aws rekognition create-stream-processor --generate-cli-skeleton
```
This command will generate a template JSON as shown below:
```sh
{
   "Name": "FacialRecognitionStreamProcessor",
   "Input": {
    "KinesisVideoStream": {
       "Arn": "<ARN of Video Stream>"
    }
   },
   "Output": {
    "KinesisDataStream": {
       "Arn": "<ARN of Data Stream>"
    }
   },
   "RoleArn": "<ARN of Role>",
   "Settings": {
    "FaceSearch": {
       "CollectionId": "faces",
       "FaceMatchThreshold": 85.5
    }
   }
}
```
Copy the content, paste it in a JSON file, fill in the necessary values and save it.

You can get the ARN of the streams and the Role either from the AWS console, or using the CLI:
```sh
# Kinesis Video
$ aws kinesisvideo list-streams
# Kinesis Data
$ aws kinesis list-streams
$ aws kinesis describe-stream --stream-name <Stream Name>
# IAM Role
$ aws iam list-roles
```

Now, let's create the stream processor by running the following command: 
```sh
aws rekognition create-stream-processor --cli-input-json file://processor.json
```
All that is returned from this command is the ARN of the Stream Processor.
check it out by running the commanf : 
```sh
$ aws rekognition list-stream-processors
# for more detailed output
$ aws rekognition describe-stream-processor --name FacialRecognitionStreamProcessor
```
Start the processor 🙏
```sh
$ aws rekognition start-stream-processor --name FacialRecognitionStreamProcessor
```
Yeahhhhh ! it works ! 🥳
to stop the prcessor, run this command:
```sh
$ aws rekognition stop-stream-processor --name FacialRecognitionStreamProcessor
```
## Step. 3
- ### Processing a Kinesis Data Stream using Lambda

The idea is that you create a function in the cloud, and execute it via some kind of input. This input can be regular HTTP requests (via an API Gateway), an Amazon SNS topic, or a Kinesis Data Stream.
There are several ways to manage your Lambda functions. You can edit in the browser or use the AWS CLI to package and upload your function. Both of these approaches are cumbersome and a better way to manage this is to use something like the Serverless Framework.

So, firs let's install Serverless
```sh
$ sudo npm install -g serverless
```
Serverless will access automatically your user via the configuration you did earlier. Redo it if necessary
```sh
export AWS_ACCESS_KEY=<YourAccessKeyId>
export AWS_SECRET_ACCESS_KEY=<YourSecretAccessKey>
export rAWS_REGION=<your-region>
```

Now, what we should do to grant permission for Serverless to do what it needs to do to deploy a function to Lambda, is to: 
* Creating S3 Buckets
* Creating new Lambda functions
* Connecting these functions to other services
* Setting up logging for this new group of services.

To do all this automatically, we need to create a _**CloudFormation**_ Stack. To allow Serverless to do all this, we need to grant our user permissions to access these services. We do this via the IAM service in the AWS Console.
Within the Policies section of IAM, click on “Create Policy” and then the JSON tab. Paste in the following:
```sh
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "logs:*",
                "cloudformation:*",
                "iam:*",
                "lambda:*"
            ],
            "Resource": "*"
        }
    ]
}
```
This Policy is allowing full access to the services required to deploy a function using Serverless.
> Note: We don not need full access permissions to do that. Just for test purposes

Finally, attach this policy to your IAM user. This will grant your user the required permissions to be able to deploy with Serverless.

Now clone this repository and do what follows:
```bash
$ git clone https://github.com/MattCollins84/pi-kinesis-video.git
$ npm install
$ sudo npm install -g typescript
```
The serverless directory contains the information required for deploying our function to Lambda. A quick overview of what each bit does:
* _**package.json**_ is used by the npm package manager to define any dependencies we may need to install
* _**tsconfig.json**_ is a simple configuration file for TypeScript
* _**serverless.yml**_ is a configuration file for our Serverless application
* _**handler.ts**_ contains the function to be called when the Lambda function is executed. It will call the processSNSEvent() method on the FacialRecognition class.
* _**lib/FacialRecognition.ts**_ is the FacialRecognition class that contains the bulk of the logic we require in our function.

A litlle explanation for what serverless.yml does:
* Defines a new stack within AWS CloudFormation called “FacialRecognitionBlog”
* Defines the use of a plugin serverless-plugin-typescript which allows us to write our Lambda functions in TypeScript
* Defines the FaaS provider as AWS. Serverless supports other providers such as Google Cloud Functions and Azure Functions
* Defines the runtime as Node.js (version 8.10). This is the language used to write our functions. Other options include Python, Ruby, Java and other popular languages. (Note: modify to node 10.x)
* Defines the AWS region we want to operate in. Remember to change this to your preferred region!
* Defines any additional permissions we may need. In this case, we want to be able to publish to an SNS Topic.
* Defines a Lambda function facialRecognition, and the file (handler) it exists in, and the actual function to call when invoking the Lambda function. handler.facialRecognition means to call the facialRecognition function in the handler.ts file.
* Creates an environment variable snsTopic that we can use as a reference within our code. This value is the ARN of an SNS Topic.
* Defines where this Lambda function will get it’s events from. In this case, it is from a Kinesis Data Stream (defined by the ARN of the stream). This means that as data comes out of the stream, it invokes the Lambda, with the event being passed to the function.

Modifu the file using our own configuration:
* the ARN of the Kinesis Data Stream
* ARN of the SNS Topic 

We can create an SNS Topic easily using the AWS CLI:
```sh
$ aws sns create-topic --name FacialRecognition
# the output
{
    "TopicArn": "<ARN of SNS Topic>"
}
```
Then subscribe to the topic created by running this command
```sh
$ aws sns subscribe --topic-arn <SNS Topic ARN> --protocol email --notification-endpoint you@yemail.com
```
A confirmation email will be sent to the address provided. Confirm it to be able to recieve notifications.

Now, when anything is published, you will recieve a notification. Run this command to verify that it works:
```sh
$ aws sns publish --topic-arn <SNS Topic ARN> --message "Hello, World!"
```
Now, let's deploy! From the serverless directory, do the following:
```sh
$ serverless deploy
```
verify that the Lambda function exists by listing all available functions:
```sh
$ aws lambda list-functions
```

Finally start streaming 
```sh
$ gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! video/x-raw,format=I420,width=640,height=480 ! x264enc bframes=0 key-int-max=45 bitrate=512 tune=zerolatency ! h264parse ! video/x-h264,stream-format=avc,alignment=au,profile=baseline ! kvssink stream-name="MyKinesisVideoStream" storage-size=512 access-key="YourAccessKey" secret-key="YourSecretKey" aws-region="YourAWSRegion"
```






#### Resources
- See: [Easily perform facial analysis on live feeds by creating a serverless video analytics environment using Amazon Rekognition Video and Amazon Kinesis Video Streams](https://aws.amazon.com/blogs/machine-learning/easily-perform-facial-analysis-on-live-feeds-by-creating-a-serverless-video-analytics-environment-with-amazon-rekognition-video-and-amazon-kinesis-video-streams/).
- Part.1: [Facial Recognition with a Raspberry Pi and Kinesis Video Streams - Part.1 ](https://medium.com/@matt.collins/facial-recognition-with-a-raspberry-pi-and-kinesis-video-streams-part-1-662f0bec5488)
- Part.2: [Facial Recognition with a Raspberry Pi and Kinesis Video Streams - Part.2 ](https://medium.com/@matt.collins/facial-recognition-with-a-raspberry-pi-and-kinesis-video-streams-part-2-9c9a631e8c24)
- [Real Time Face Identification on Live Camera Feed using Amazon Rekognition Video and Kinesis Video Streams](https://medium.com/zenofai/real-time-face-identification-on-live-camera-feed-using-amazon-rekognition-video-and-kinesis-video-52b0a59e8a9)