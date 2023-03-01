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

**"echo" Operation TEST**
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


**"create" Operation TEST**
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

3. Click the arrow on the "Test" button and select the saved event we just created called "create" [**Note this will not work if your DynamoDB table doesn't have the right table name or if it isn't created properly - View Step 1**]

![Successful Test - 200 Code](https://user-images.githubusercontent.com/126350373/221986583-687f611d-f9ca-4753-baca-97f20da26695.png)

4. (Optional) Go to the DynamoDB service and click "Tables" on the left-hand panel. Next click the DynamoDB table we created in step 1 called "apigateway-lambda-crud". After that, click on "Explore items" on the left-hand panel. Lastly, you should see a new item created from our "Create Operation Test".

![Verified "Create Operation Test"](https://user-images.githubusercontent.com/126350373/221987329-71ddaf06-654b-476d-911b-71106e764aa5.png)


We are all set to create our API in API Gateway console

### Step 5 - Create API

**To create the API**
1. Go to API Gateway console
2. Scroll down to REST API and click "Build"

![Build API](https://user-images.githubusercontent.com/126350373/221991046-d4a3f54c-5f82-48c9-9941-4cf100559302.png)

3. Make sure to select "New API" and Give the API name as **DynamoOperations**, keep everything as is, click "Create API"

![Create REST API](https://user-images.githubusercontent.com/126350373/221992297-49184253-380d-4ee7-afb3-cec597b67dd7.png)

4. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next 

Click "Actions", then click "Create Resource"

![Create API resource](https://user-images.githubusercontent.com/126350373/222020359-db68ce6e-65ca-4eee-ae8b-b6d65351ee21.png)

5. Input **DynamoOperations** in the Resource Name, Resource Path will get populated. Click "Create Resource"

![Create resource](https://user-images.githubusercontent.com/126350373/221992928-d7646703-bda8-477e-bb92-a71a9834db7d.png)

6. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Actions" again and click "Create Method". 

![Create resource method](https://user-images.githubusercontent.com/126350373/221993094-1fa244ec-91c9-4c74-a308-d0c927604681.png)

7. Select "POST" from drop down , then click checkmark

![Create resource method](https://user-images.githubusercontent.com/126350373/221993423-c955c2e1-7f6c-40ce-82aa-7062359c5679.png)

8. The integration will come up automatically with "Lambda Function" option selected. Select **LambdaCRUDOverHTTPS** function that we created earlier. As you start typing the name, your function name will show up. Select and click "Save". A popup window will come up to add resource policy to the lambda to be invoked by this API. Click "Ok"

![Create lambda integration](https://user-images.githubusercontent.com/126350373/221993769-f5291680-0647-492b-883b-0fdbe2a13891.png)

Our API-Lambda integration is done!

### Step 6 - Deploy the API

In this step, you deploy the API that you created to a stage called prod.

1. Click "Actions", select "Deploy API"

![Deploy API](https://user-images.githubusercontent.com/126350373/222020622-f3317982-5d0a-492b-a418-235350ee05d4.png)

2. Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"

![Deploy API to Prod Stage](https://user-images.githubusercontent.com/126350373/221994238-db3d8c5a-8102-4a9d-bb6d-34a6e6cca617.png)

3. We're all set to run our solution! To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen (we are going to need it in Step 7)

![Copy Invoke URL](https://user-images.githubusercontent.com/126350373/222017833-9a7e2780-dd44-4ce9-afa2-17c6bc5b2fdd.png)


### Step 7 - Running our solution

1. Go to Postman.com and go to your workspace. (If you've never used Postman then you will have to sign up, no worries it is free and you can use the browser interface)

![Select Postman Workspace](https://user-images.githubusercontent.com/126350373/221995121-5e57ae27-b04f-44ab-b73a-04beac313a56.png)

2. Click the "New" button then select HTTP Request as shown in the screenshot below.
![Creating HTTP Request](https://user-images.githubusercontent.com/126350373/221995255-aef9b50d-de32-4774-b1a1-4c7b80cbfd14.png)

3. Switch "GET" to "POST". Next, copy and paste the Invoke URL that we copied from our API Gateway in the previous step. Lastly, select "Body" then the "raw" option. All shown in the screenshot below.

![Postman Setup](https://user-images.githubusercontent.com/126350373/222018220-66eb3ea8-da1c-4c46-af8f-823c56ed6cea.png)

4. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "create",
    "tableName": "apigateway-lambda-crud",
    "payload": {
        "Item": {
            "id": "ABCD",
            "number": 879
        }
    }
}
```
5. Paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.
![Successful Postman Request - 200 Code](https://user-images.githubusercontent.com/126350373/222018691-1934dd6e-e171-495b-9dee-39b65edca622.png)

 
6. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select **apigateway-lambda-crud** table, select "Explore items" tab, and the newly inserted item should be displayed.

![Dynamo Item](https://user-images.githubusercontent.com/126350373/222018953-d3c4f4d2-e835-41a4-94e7-db1b954a776c.png)

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Cleanup

Let's clean up the resources we have created for this lab.


### Step 8 - Cleaning up DynamoDB, Lambda, and API Gateway

To delete the table, from DynamoDB console, select the table "apigateway-lambda-crud", and click "Delete table"

![Delete Dynamo](https://user-images.githubusercontent.com/126350373/222019212-453cf01d-36ad-4f38-bf36-4c1988ad84ef.png)

To delete the Lambda, from the Lambda console, select lambda "LambdaCRUDOverHTTPS", click "Actions", then click Delete 

![Delete Lambda](https://user-images.githubusercontent.com/126350373/222019298-937dc8ef-49ed-4242-88ca-5c9072bf5924.png)

To delete the API we created, in API gateway console, under APIs, select "DynamoOperations" API, click "Actions", then "Delete"

![Delete API](https://user-images.githubusercontent.com/126350373/222019458-0c6d6050-4f39-4d7a-8a68-6fa2bb64bef9.png)
