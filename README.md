# serverless-API-using-API-Gateway-Lambda-and-DynamoDB

High Level Design
![Presentation1](https://github.com/user-attachments/assets/d9d3bee0-4cf0-4f34-a25c-0719ce3c7a55)

This serverless API demonstrates how AWS Lambda, API Gateway, and DynamoDB can be used to build scalable and cost-effective microservices. While this is currently a standalone function, the same approach can be extended to design fully decoupled microservices that handle different business logic independently.

An Amazon API Gateway is a collection of resources and methods. For this tutorial, you create one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). That is, when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

* Create, update, and delete an item.
* Read an item.
* Scan an item.
* Other operations (echo, ping), not related to DynamoDB, that you can use for testing.

The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:
```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
```
The following is a sample request payload for a DynamoDB read item operation:
```
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```
## Setup
### Create Lambda IAM Role
Create the execution role that gives your function permission to access AWS resources.

To create an execution role

1. Open the roles page in the IAM console.
2. Choose Create role.
3. Create a role with the following properties.
   * Trusted entity – Lambda.
   * Role name – lambda-apigateway-role.
   * Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that the function needs to write data to DynamoDB and upload logs.
```
{
"Version": "2012-10-17",
"Statement": [
{
  "Sid": "Stmt1428341300017",
  "Action": [
    "dynamodb:DeleteItem",
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:UpdateItem"
  ],
  "Effect": "Allow",
  "Resource": "*"
},
{
  "Sid": "",
  "Resource": "*",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Effect": "Allow"
}
]
}
```
## Create Lambda Function
### To create the function

1. Click "Create function" in AWS Lambda Console

2. Select "Author from scratch". Use name LambdaFunctionOverHttps , select Python 3.7 as Runtime. Under Permissions, select "Use an existing role", and select lambda-apigateway-role that we created, from the drop down

3. Click "Create function"

4. Replace the boilerplate coding with the following code snippet and click "Deploy"

Example Python Code

```
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
```

## Test Lambda Function
Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.

1. Click the arrow on "Select a test event" and click "Configure test events"
2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output. Click "Create" to save
```
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
```
3. Click "Test", and it will execute the test event. You should see the output in the console

We're all set to create DynamoDB table and an API using our lambda as backend!

## Create DynamoDB Table

### Create the DynamoDB table that the Lambda function uses.

To create a DynamoDB table

1. Open the DynamoDB console.
2. Choose Create table.
3. Create a table with the following settings.
   * Table name – lambda-apigateway
   * Primary key (partition key) – id (string)
4. Choose Create.

## Create API
### To create the API
1. Go to API Gateway console
2. Click Create API
3. Scroll down and select "Build" for REST API
4. Give the API name as "DynamoDBOperations", keep everything as is, click "Create API"
5. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next
6. Click "Actions", then click "Create Resource"
7. Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"
8. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Actions" again and click "Create Method".
9. Select "POST" from drop down , then click checkmark
10. The integration will come up automatically with "Lambda Function" option selected. Select "LambdaFunctionOverHttps" function that we created earlier. As you start typing the name, your function name will show up.Select and click "Save". A popup window will come up to add resource policy to the lambda to be invoked by this API. Click "Ok"

Our API-Lambda integration is done!

## Deploy the API

In this step, you deploy the API that you created to a stage called prod.

1. Click "Actions", select "Deploy API"
2. Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"
3. We're all set to run our solution! To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen

## Running our solution
1. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:
```
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
```
2. To execute our API from local machine, we are going to use Postman and Curl command. You can choose either method based on your convenience and familiarity.
   * To run this from Postman, select "POST" , paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.
3. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select "lambda-apigateway" table, select "Items" tab, and the newly inserted item should be displayed.
4. To get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table
```
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```
We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Load Tesing & Measuring Performance and Cost

Initial Lambda Setting

<img width="778" alt="image" src="https://github.com/user-attachments/assets/b1dc52a1-28b9-44e7-8670-206c2168e158" />

Load Test

<img width="360" alt="image" src="https://github.com/user-attachments/assets/e03d1161-b8cc-46c3-bad0-670375b9dc86" />

Initial Results

Performance

<img width="573" alt="image" src="https://github.com/user-attachments/assets/835b84b8-3c37-4bfc-8753-fe258e4426d2" />
<img width="569" alt="image" src="https://github.com/user-attachments/assets/dea649bc-3ed9-4df9-9fd5-b1b98a50aa65" />

Cost

<img width="535" alt="image" src="https://github.com/user-attachments/assets/c91ea915-c35d-4e19-ba02-dc82712d794d" />

Updated Lambda Setting

<img width="773" alt="image" src="https://github.com/user-attachments/assets/6ec0bb91-ddab-4fda-be5d-bc7b15e09356" />

Improved Results

Performance

<img width="575" alt="image" src="https://github.com/user-attachments/assets/cc0513d8-71e4-4311-8625-34ed9747ccfe" />
<img width="577" alt="image" src="https://github.com/user-attachments/assets/97e02f8e-918c-4172-bab3-f32a076e3a10" />

Cost

<img width="525" alt="image" src="https://github.com/user-attachments/assets/cd66a2d6-ffe5-4979-babf-b1e348f2c9a8" />

## Conclusion

* The tests revealed that increasing Lambda memory allocation significantly improved throughput and response times, making the application more responsive under load. However, this improvement came at the cost of higher AWS expenses, highlighting the trade-off between performance and cost in serverless architectures.

* Optimizing serverless applications requires careful balancing of resource allocation to achieve the desired performance while keeping costs under control. Higher memory allocation can be beneficial for latency-sensitive applications, whereas lower memory configurations may be more suitable for cost-conscious workloads with less stringent performance requirements.

* Ultimately, the ideal configuration depends on factors such as workload characteristics, request patterns, budget constraints, and performance expectations. Continuous monitoring, testing, and fine-tuning are essential to achieving an optimal balance between efficiency and cost-effectiveness in a serverless environment.







