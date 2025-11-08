# tune-it-up
![tune-it-up-serverless](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/tune-it-up-serverless.png)

An API Gateway is a collection resources and methods. We create one resource (DynamoDBManager) and define one method (POST) right on it. We back this up with a Lambda function (LambdaFunctionOverHttps). When calling the API through an HTTPS endpoint, Amazon API Gateway invokes that Lambda Function.

### Supported operations with the POST method on our DynamoDBmanager resource
- **C**reate, **R**ead, **U**pdate and **D**elete an item. | **CRUD**
- Scan an item.
- Other operations -> echo & ping. While NOT related to DynamoDB, it can be useful for testing.

When sending that **POST** request, it takes care of identifying the **DynamoDB operation** and provides back the data.

Here's a sample request payload for a *create* item operation on DynamoDB:
```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "43049",
            "name": "Asgardian"
        }
    }
}
```

Here's another sample request payload for DynamoDB but for a *read* item operation:
```json
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

### Now, it's your turn!

Just like in the video, you will replicate my steps to witness the power of tuning!

### Create Custom Policy
Get a custom policy going with the least privilege in mind.

A) Open policies page in the IAM console  
B) Click **Create policy** on top right corner  
C) In the policy editor, click JSON, and paste:  

```json
{
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "AsgardianCRUD",
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
![custom-policy](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/custom-policy.png)
D) Name it "lambda-custom-policy" & click **Create Policy** on bottom right

### Create Lambda IAM role
*This is the IAM role that lets your function access required AWS resources.*

To create an execution role:

A) Open the **roles page** in the IAM console  
B) Choose **Create role**  
C) Create a role with the following properties  
  - Trust entity type **>** AWS service, now select Lambda from Use case  
  - Permissions **>** In your permission policies page, in the search bar, type **lambda-custom-policy**. That newly create     policy should now show up. Select it, and click next.
  
![permissions](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/custom-policy.png)

- Role name **>** lambda-apigateway-role
- Click **Create role**
  
*To create that function*  
A) Click **Create function** in AWS Lambda console

![create-function](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/create-function.png)

B) Select **Author from scratch**. Use name **LambdaFunctionOverHttps**. Select **Python 3.13** as Runtime. Under *Permissions* click the arrow beside *Change default execution role*, and use *an existing role*, then select **lambda-apigateway-role** that we created, from the drop down menu
C) Click **Create function**
D) Replace the default code with below's snippet and click **Deploy**

**Python snippet code**
```python
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

![python-snippet](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/python-snippet.png)

### Test your Lambda Function

Let's give that Lambda function a test. We'll simply do a sample echo operation for now. Once DYnamoDB and the API are up, we can go deeper. The function **should** output whatever input you pass.

A) Click the **Test** tab right beside **Code** tab
B) Give **Event name** as **echotest**
C) Paste below's JSON into the event. The field *operations* instructs what the lambda function will perform. In our case, it will return the payload from our input event as output. Click *Save* to save.

```json
{
    "operation": "echo",
    "payload": {
        "somekey1": "lokivalue1",
        "somekey2": "thorvalue2"
    }
}
```
D) Click **Test* and it will execute your test event. You should see the output in the console.  

![test-lambda-function](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/test-lambda-function.png)

Buckle up. We can now create DynamoDB table and an API using our lambda as backend!

### Create your DynamoDB table
*Create the DynamoDB table that the Lambda function uses.

***Steps to create***
A) Open the DynamoDB console
B) Choose *tables* from the left pane, now click *Create table* on top right.
C) Create a table with settings:  

- Table name **>** lambda-apigateway
- Partition key **>** id (string)
D) Choose *Create table*  

![dynamodb-table](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/dynamodb-table.png)

### Create the API
A) Go to API Gateway console
B) Click create API
C) Scroll down and select *Build for REST API*
D) Name it **DynamoDBOperations**, keep the rest as is and click *Create API*

![create-rest-api](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/create-rest-api.png)

E) Each API is a collection of resources and methods which are ingested with backend HTTP endpoints. Lambda functions, or other AWS services. Usually, API resources are organized in a resource tree based on the application logic. In our scenario, we only have the root resource. Let's spice it up a bit by adding a resource. Click *Create Resource*.
F) Input **DynamoDBManager* in the Resource Name. Now click *Create Resource*

![create-method](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/create-method.png)

G) Select *POST* from drop down
H) *Integration type* should be pre-selected with *Lambda Function*. Select *LambdaFunctionOverHttps*, the function we created earlier. Type the name of the function, and it should appear. Select it **>** scroll down **>** and click *Create method*.

![post-method](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/post-method.png)

The API Lambda integration is done! Nice.

### Deploy the API

We can now deploy the API we created to a stage called *Prod*.

A) Click *Deploy API* on your top right
B) Select *New stage*. Name it **Prod** under *Stage name*. Click *Deploy

![deploy-api])(https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/deploy-api.png)

C) We're ready to run that solution! We need that endpoint URL to invoke the API endpoint. In the *Stages* screen, expand the *Prod* until you see *POST. Now Select *POST* method, and copy the *Invoke URL*.

![invoke-url](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/invoke-url.png)

### Test our solution

A) Lambda function supports the *Create* operation to create an item in your DynamoDB table. You can request this operation using the following JSON:

```json
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
B) Now, to execute our API from a **local machine**, we're going to use **Postman** and a Curl command. Choose your preferred method. Either way works!

- To run this from Postman, select **POST**, paste that API invoke URL. Now, under **Body**, select **raw** and paste our JSON from above. Click **Send**. The API should execute and return **HTTPStatusCode** 200.

![postman](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/postman.png)

- To run this from a terminal, using cURL:
```
$ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
```
C) Validate the item is indeed inserted into your DynamoDB table. Go to **Dynamo** console, select *lambda-apigateway* table, select *Explore table items* button from your top right, and the newly inserted item should be displaying!

![explore-table](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/explore-table.png)

![table-view](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/table-view.png)

D) To get all the inserted items from the table, we can use the *list* operation of Lambda using that same API. Pass the following JSON to the API, and it will return all the items from the DynamoDB table.

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```
![postman-table](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/postman-table.png)

Cleanup time!

### Cleaning up DynamoDB

- To delete the table from DynamoDB console, select the table *lambda-apigateway. Now, top right, click *Actions*, then *Delete table*
- To delete your Lambda, from the Lambda console, select Lambda *LambdaFunctionOverHttps*, click *Actions, then click *Delete*
- To delete the API we created, in *API Gateway console, under *APIs, select *DynamoDBOperations* API, click *Delete*





