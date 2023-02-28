# AWS Serverless API Lab

## Lab Overview And High Level Design

In this lab we will allow the client to perform an API call (or request) through our API Gateway (via a HTTPS endpoint). Our API gateway will then trigger a lambda function that is coded to perform CRUD (Create, Read, Update, and Delete) operations into our DynamoDB table.

**High Level Design - Serverless API Architecture**

![Microservice API Gateway (1)](https://user-images.githubusercontent.com/126350373/221657043-ddfe1ce8-3194-4e4e-ba06-cd146d2d7467.png)


The client will perform the API call by using the "POST" HTTP method. Our client must provide a request payload to perform an API call. The request payload identifies the DynamoDB operation (CRUD) the client wants to perform with the necessary data. 

*Assuming **apigateway-lambda-crud** is the table name of our DynamoDB table.*

An example request payload for a CREATE opertion shows as follows:

```json
{
    "operation": "create",
    "tableName": "apigateway-lambda-crud",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Sam"
        }
    }
}
```

An example request payload for a DELETE operation shows as follows:

```json
{
    "operation": "delete",
    "tableName": "apigateway-lambda-crud",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Sam"
        }
    }
}
```

An example request payload for a READ operation shows as follows:

```json
{
    "operation": "read",
    "tableName": "apigateway-lambda-crud",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```

## Setup

### Step 1 - Create DynamoDB Table

Create the DynamoDB table that the Lambda function uses.

**To create a DynamoDB table**

1. Open the DynamoDB console.
2. Choose Create table.
3. Create a table with the following settings.
   * Table name – **apigateway-lambda-crud**
   * Primary key – id (string)
4. Choose Create.

![create DynamoDB table](https://user-images.githubusercontent.com/126350373/221652327-b62fe492-8776-4b2f-84ed-8765a97015f6.png)


### Step 2 - Create Lambda IAM Role 
Create the execution role that gives your function permission to access AWS resources.

To create an execution role

1. Open the roles page in the IAM console.
2. Choose Create role.
3. Create a role with the following properties.
    * Trusted entity – Lambda.
    * Role name – **lambda-apigateway-dynamodb-role**.
    * Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that  the function needs to write data to DynamoDB and upload logs. 
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem",
                "dynamodb:GetItem",
                "dynamodb:Scan",
                "dynamodb:Query",
                "dynamodb:UpdateItem",
                "logs:PutLogEvents",
                "logs:CreateLogGroup"
            ],
            "Resource": "*"
        }
    ]
}
    ```

### Step 3 - Create Lambda Function

**To create the function**
1. Click "Create function" in AWS Lambda Console

![Create function](https://user-images.githubusercontent.com/126350373/221980515-da86c483-004e-449b-93f6-5f9388e959e8.png)


2. Select "Author from scratch". Use name **LambdaCRUDOverHTTPS** , select **Python 3.7** as Runtime. Under Permissions, select "Use an existing role", and select **lambda-apigateway-dynamodb-role** that we created, from the drop down

3. Click "Create function"

![Lambda basic information](https://user-images.githubusercontent.com/126350373/221984406-df5c15dc-1c36-4bb8-81e5-ea9986025a9d.png)

4. Replace the existing sample code with the following code snippet and click "Deploy"

**Python CRUD Operation Code**
```python
import boto3
import json

print('Loading function')

def lambda_handler(event,context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    
    print("Received event: " + json.dumps(event, indent=1))
     
    operation = event['operation']
    

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
    #CRUD operations shown below:
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'echo': lambda x: x
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
        
```
![Lambda Code](https://user-images.githubusercontent.com/126350373/221973523-c46347aa-2dfb-47d0-874e-a8c55d1c2456.png)

### Step 4 - Test Lambda Function

Let's test our newly created function. We will test our **"echo"** operation AND our **"create"** operation. **"echo"** is an operation that should output whatever the request/input/event submitted. **"create"** is an operation that will create a real record into the DynamoDB table (*apigateway-lambda-crud*) we created in the first step.  

**Echo Operation TEST**
1. Click the arrow on the "Test" button and click "Configure test events"

![Configure test events](https://user-images.githubusercontent.com/126350373/221981236-6632840f-3f45-4373-9e2c-f1644699b67a.png)

2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output. Click "Create" to save.
```json
{
  "operation": "echo",
  "payload": {
    "testkey1": "testvalue1",
    "testkey2": "testvalue2"
  }
}
```
![Save test event](https://user-images.githubusercontent.com/126350373/221981831-b5ad9171-fb46-4747-a90e-a6ec0f5f168c.png)


3. Click "Test", and it will execute the test event. You should see the output in the console

![Successful Test](https://user-images.githubusercontent.com/126350373/221988503-418d8ea0-a8b1-452c-badb-379bf3456d3c.png)


**Create Operation TEST**
1. Click the arrow on the "Test" button and click "Configure test events"

![Configure test events](https://user-images.githubusercontent.com/126350373/221985018-da9da98e-6298-4af1-a255-474d14b31837.png)

2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it will create a new record into our DynamoDB table. Click "Create" to save.
```json
{
  "operation": "create",
  "tableName": "apigateway-lambda-crud",
  "payload": {
    "Item": {
      "id": "ABC",
      "number": 5,
      "name": "Bob",
      "age": 42
    }
  }
}
```
![Save test event](https://user-images.githubusercontent.com/126350373/221985916-1652e2b6-dd0e-4ef3-b997-85e25aaa9dd2.png)

3. Click the arrow on the "Test" button and select the saved event we just created called "create" [**Note this will not work if your DynamoDB table doesn't have the right table name or if it isn't created properly - View Step 1"**]

![Successful Test - 200 Code](https://user-images.githubusercontent.com/126350373/221986583-687f611d-f9ca-4753-baca-97f20da26695.png)

4. (Optional) Go to the DynamoDB service and click "Tables" on the left-hand panel. Next click the DynamoDB table we created in step 1 called "apigateway-lambda-crud". After that, click on "Explore items" on the left-hand panel. Lastly, you should see a new item created from our "Create Operation Test".

![Verified "Create Operation Test"](https://user-images.githubusercontent.com/126350373/221987329-71ddaf06-654b-476d-911b-71106e764aa5.png)


We are all set to create our API in API Gateway console

### Step 5 - Create API

**To create the API**
1. Go to API Gateway console
2. Scroll down to REST API and click "Build"

![create API](./images/create-api-button.jpg) 
![Build API](https://user-images.githubusercontent.com/126350373/221991046-d4a3f54c-5f82-48c9-9941-4cf100559302.png)

3. Scroll down and select "Build" for REST API

![Build REST API](./images/build-rest-api.jpg) 

4. Give the API name as "DynamoDBOperations", keep everything as is, click "Create API"

![Create REST API](./images/create-new-api.jpg)

5. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next 

Click "Actions", then click "Create Resource"

![Create API resource](./images/create-api-resource.jpg)

6. Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"

![Create resource](./images/create-resource-name.jpg)

7. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Actions" again and click "Create Method". 

![Create resource method](./images/create-method-1.jpg)

8. Select "POST" from drop down , then click checkmark

![Create resource method](./images/create-method-2.jpg)

9. The integration will come up automatically with "Lambda Function" option selected. Select "LambdaFunctionOverHttps" function that we created earlier. As you start typing the name, your function name will show up.Select and click "Save". A popup window will come up to add resource policy to the lambda to be invoked by this API. Click "Ok"

![Create lambda integration](./images/create-lambda-integration.jpg)

Our API-Lambda integration is done!

### Step 6 - Deploy the API

In this step, you deploy the API that you created to a stage called prod.

1. Click "Actions", select "Deploy API"

![Deploy API](./images/deploy-api-1.jpg)

2. Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"

![Deploy API to Prod Stage](./images/deploy-api-2.jpg)

3. We're all set to run our solution! To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen

![Copy Invoke Url](./images/copy-invoke-url.jpg)


### Step 7 - Running our solution

1. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:

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
2. To execute our API from local machine, we are going to use Postman and Curl command. You can choose either method based on your convenience and familiarity. 
    * To run this from Postman, select "POST" , paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.

    ![Execute from Postman](./images/create-from-postman.jpg)

    * To run this from terminal using Curl, run the below
    ```
    $ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
    ```   
3. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select "lambda-apigateway" table, select "Items" tab, and the newly inserted item should be displayed.

![Dynamo Item](./images/dynamo-item.jpg)

4. To get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```
![List Dynamo Items](./images/dynamo-item-list.jpg)

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Cleanup

Let's clean up the resources we have created for this lab.


### Step 8 - Cleaning up DynamoDB

To delete the table, from DynamoDB console, select the table "lambda-apigateway", and click "Delete table"

![Delete Dynamo](./images/delete-dynamo.jpg)

To delete the Lambda, from the Lambda console, select lambda "LambdaFunctionOverHttps", click "Actions", then click Delete 

![Delete Lambda](./images/delete-lambda.jpg)

To delete the API we created, in API gateway console, under APIs, select "DynamoDBOperations" API, click "Actions", then "Delete"

![Delete API](./images/delete-api.jpg)
