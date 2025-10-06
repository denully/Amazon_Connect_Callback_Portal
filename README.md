<img width="525" height="550" alt="image" src="https://github.com/user-attachments/assets/ab1bdc73-fc58-47bb-b658-7acca6805ce2" />

# AWS Callback Request

## Getting started

To set this up, you will need access to the following: 
- Amazon Connect
- AWS Lambda
- API Gateway

## Amazon Connect
Create a Contact flow (inbound), 
A very basic test flow can be found here, [template here](https://github.com/denully/Amazon-Connect_Callback/blob/main/Callback_Form.json)
 - Start
  - Set logging behavior
   - Check Contact Attributes: User defined/$.Attributes.recordingConsent, Equals: YES,  Equals: NO
    - YES: Set recording and analytics behavior, Turned on.
    - NO: Set recording and analytics behavior, Turned Off.
     -Transfer to queue:  Transfer to callback, Set working queue: your desired queue.
      - Disconnect.
Publish the Flow and go to About this flow and copy the Instance ID And the flow ID. 
(arn:aws:connect:REGION:ACCOUNT:instance/**INSTANCE ID**/contact-flow/**FLOW ID**)
Go to your Queue overview and find the queue you use and get the Queue ID under Show Additional queue information.
These ID's will be used in the lambda code in the next step.

## AWS Lambda

This will be the script that pulls the data from the web form, to the amazon connect flow.
In AWS Lambda:
 - Create function
 - Author from scratch
 - Function name: give it a suited name of your choosing.
 - Runetime: Python
 - Architecture: x86_64
 - Permissions, expand the Change default execution role and create a new role (or use exising if you have specific ones setup)
    - Permissions needed: 
    ```
    logs:CreateLogGroup
    logs:CreateLogStream
    logs:PutLogEvents
    ec2:CreateNetworkInterface
    ec2:DeleteNetworkInterface
    ec2:DescribeNetworkInterfaces
    connect:StartOutboundVoiceContact
    connect:DescribeInstance
    connect:ListQueues
    connect:ListContactFlows
    ```
    An example could be as below, however be aware that this is setup to give the access to all your instance, it is best to have it set for specific instances and resources
    ```
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces".
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "connect:StartOutboundVoiceContact",
                "connect:DescribeInstance",
                "connect:ListQueues",
                "connect:ListContactFlows"
            ],
            "Resource": "*"
        }
     ]
    }
    ```
 - Create function.
 - Under Code in the newly created Lambda, use the code from the [lambda_function.py](https://github.com/denully/Amazon-Connect_Callback/blob/main/lambda_function.py)
 - Under Code you will need to edit the ID's for Amazon Connect.
 ```
         response = connect.start_outbound_voice_contact(
            DestinationPhoneNumber='+XXXXXXXXXX',  # AWS Number assigned to the callback Queue. This is needed as AWS connect.start_outbound_voice_contact always calls the number right away, so we lead them into the flow first where we change the number to the customer to receive a callback.
            ContactFlowId='XXXXXXXXXXXX',  # Your callback flow ID (Found under About this Flow, inside the Amazon Connect flow)
            InstanceId='XXXXXXXXXXXX',     # Your Instance ID (Can also be found under About this Flow, inside the Amazon Connect flow, right before the flow ID)
            QueueId='XXXXXXXXXXXX',        # Your callback queue ID (Found under Show additional queue information, inside the Amazon Connect queue)
            Attributes={
                'CustomerPhone': customer_phone,  # Real customer number
    ........
 ```
 - Deploy (CTRL+SHIFT+U)
 - Go under Actions and Publish new version
Copy the Function ARN link for later use in the API Gateway (can also be searched in a drop down)


## API Gateway

This will be the bridge between the site and the Lambda function.
In AWS API Gateway:
 - Create a REST API
 - Give it a name
 - API endpoint type: Regional
 - IP address type: IPv4
 Click Create API. 

 Once the API is setup, click it to open it up, click Create Resource (left hand side).
 - Give it a resource name e.g. callback, this would make the endpoint look like this: https://abc123.execute-api.us-east-1.amazonaws.com/prod/callback
 - Enable CORS (Cross Origin Resource Sharing)
 Create Resource

 Select the resource in the right hand side, should look like /callback (or the name you gave it)
 - in the right hand side click Create method
 - Methode type: POST (from the drop down)
 - Integration type: Lambda function
 - Lambda Proxy integration: Enable this
 - Lambda function: chose your region and paste in the Lambda ARN created earlier or find it in the drop down.
 Click Create Methode

Again select the resource e.g /callback, then select Enable CORS in the top right corner.
 - Access Control Allow Methods: POST and Options
 - Access Control Allow Headers: Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token
 - Access Control Allow Origin: *
Save
Deploy API
 - Deployment stage: New stage
 - Stage name: e.g. prod
Deploy and copy the API URL, It looks like: https://abc123def.execute-api.us-east-1.amazonaws.com/prod , Your callback endpoint is: https://abc123def.execute-api.us-east-1.amazonaws.com/prod/callback

## Web form (HTML)

In the html code in [Req_callback.html](https://github.com/denully/Amazon-Connect_Callback/blob/main/Req_callback.html), which can be renamed without any issues.
find the section with: 
```
// Enter in the path for the API created in AWS API Gateway
    try {
      const response = await fetch('ENTER API PATH HERE', {
        method: 'POST',
.......
```
And replace the "ENTER API PATH HERE" with the link for the API, keeping the beginning and ending '

## Test and usage.

If you enter a number that does not match the countries number format, it will say invalid number, e.g. in Denmark we have +45 as country code, followed by 8 digits.
So typing in +451234567 will fail, where +4512345678 will succeed.  It does not validate if it is an active number prior to submitting the request.

Enter a valid number and tick off if you allow recording or not and press Submit request.
Login to your Amazon Connect CCP (Contact control panel) and make yourself available, if you are part of the queue used for callbacks.
Shortly after (if not other calls are in the queue) you will receive a call that shortly after will call the number submitted.

## DISCLAIMER !!!!

I am not a professional programmer in any way, this was made by using AWS documentation, my experience with Amazon Connect and the depending AWS services, some HTML from old webdesigner days, as well as some ChatGPT 5.0. :) 
So i am sure this can be optimized and coding can be minimized, but it is something to develop on and i will also update as i learn more. 

## Roadmap.

No specific roadmap is planned, but my hopes is to add the below at some point.
- captcha
- SMS notifications
- Cancellation of callback (this will require a different setup though, with DynamoDB, as the current setup cant cancel a callback once it enters the connect queue.
