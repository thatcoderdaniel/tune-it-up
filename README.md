# tune-it-up
![tune-it-up-serverless](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/tune-it-up-serverless.png)

An API Gateway is a collection resources and methods. We create one resource (DynamoDBManager) and define one method (POST) right on it. We back this up with a Lambda function (LambdaFunctionOverHttps). When calling the API through an HTTPS endpoint, Amazon API Gateway invokes that Lambda Function.

### Supported operations with the POST method on our DynamoDBmanager resource
- **C**reate, **R**ead, **U**pdate and **D**elete an item. | **CRUD**
- Scan an item.
- Other operations -> echo & ping. While NOT related to DynamoDB, it can be useful for testing.

When sending that **POST** request, it takes care of identifying the **DynamoDB operation** and provides back the data.

Here's a sample request payload for a *create* item operation on DynamoDB:
```
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

### Now, it's your turn!

Just like in the video, you will replicate my steps to witness the power of tuning!

### Create Custom Policy
Get a custom policy going with the least privilege in mind.

A) Open policies page in the IAM console  
B) Click **Create policy** on top right corner  
C) In the policy editor, click JSON, and paste:  

```
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
  - Permissions **>** In your permission policies page, in the search bar, type **lambda-custom-policy**. That newly create     policy should now sho up. Select it, and click next.
  
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

![python-snippet](https://github.com/thatcoderdaniel/tune-it-up/blob/main/images/python-snippet.png)

