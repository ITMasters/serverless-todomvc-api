## Setup
```shell
serverless create --template aws-nodejs --path todomvc-api
cd todomvc-api
serverless deploy
```

If you've configured serverless correctly you should see a message that ends in
```
Serverless: Stack update finished...
Service Information
service: todomvc-api
stage: dev
region: us-east-1
api keys:
  None
endpoints:
  None
functions:
  hello: todomvc-api-dev-hello
```

You can then run

```shell
serverless invoke --function hello
```

And you should see a response saying
```json
{
    "statusCode": 200,
    "body": "{\"message\":\"Go Serverless v1.0! Your function executed successfully!\",\"input\":{}}"
}
```

We're going to need to aws-sdk node package and another package called uuid, so let's go ahead and install them and save them as dependencies 

```shell 
npm init -f
npm install --save-dev aws-sdk 
npm install --save uuid
```
## Create
In a new directory called todos, create a new javascript file called create.js.
There's lots of useful in the default serverless.yaml file but we don't need any of it now so you can go ahead and delete anything that's commented out.
We want have our new create.js file to handle the create requests so let's change our function name to create and our handler to todos.create. Your serverless.yml file should now look like this:
```
service: todomvc-api

provider:
  name: aws
  runtime: nodejs6.10

functions:
  create:
   handler: todos/create.create
```

For now let's just copy the code from handles.js into create.js and change the function name to create and the message to something that let's us see that 
```javascript
'use strict';

module.exports.create = (event, context, callback) => {
    const response = {
        statusCode: 200,
        body: JSON.stringify({
            message: 'This is the create function',
            input: event,
        }),
    };

    callback(null, response);
};
```
Now if we run `serverless deploy` again from the home directory we should see that the stack has been updated and our create function deployed.

Run `serverless invoke --function create` to check that everything is working.
```json
{
    "statusCode": 200,
    "body": "{\"message\":\"This is the create function\",\"input\":{}}"
}
```
You might have noticed that in that response there's an empty `input` variable and that in our handler code this corresponding to an argument called event.
When using the serverless command line we can pass data directly to a function using the --data flag

You can also use the sls command as shorthand for the serverless command.

```shell
sls invoke --function create --data "Hello World"
```
Will give the response
```json
{
    "statusCode": 200,
    "body": "{\"message\":\"This is the create function\",\"input\":\"Hello World\"}"
}
```
As you can see it's easy to get a lambda function set up and running with the serverless framework. But we don't want to use the command line to call our function, we want to use it as part of the API for our single page app. Do to that we'll need to add an http endpoint. Fortunately, this is easy too. Open open `serverless.yml` and add the following lines so your functions section looks like this:
```yaml
functions:
  create:
    handler: todos/create.create
    events:
      - http:
          path: todos
          method: post
          cors: true
```
We only added 5 lines but it did a lot for us. We added a new `events` property to our create functions, which is a list of the events that should trigger this function.
We added one type of event - an http request on the path /todos. This functions deals with creating a Todo, which should be done via a POST request so we only handle that request type. Finally, because our API won't be hosted on the same domain as the frontend of our applications, we set cors to true to allow cross origin requests.
This is all we need to do to get a function http endpoint deployed, so let's go ahead and deploy it and test it out.

```shell
sls deploy 
...
endpoints:
  POST - https://980urz0ll1.execute-api.us-east-1.amazonaws.com/dev/todos
...
```
Note the url of your endpoiint will be different as it is unique to your project, but it will still end in /dev/todos. The /todos is the path we specified for our endpoint while the /dev is the stage. We haven't specificied a stage in our `serverless.yml` file so serverless uses the default, which is dev.
Let's test our endpoint out! You can could an application like [Postman](https://www.getpostman.com/) which I use a lot for testing APIs to do this, but we'll use the command line for now. 

Replace the url with the endpoint you got from the command above
```shell
curl --request POST https://yoururl/dev/todos
```

You should get back a fairly long looking json response, which when prettified looks like this:
```json
{  
   "message":"This is the create function",
   "input":{  
      "resource":"/todos",
      "path":"/todos",
      "httpMethod":"POST",
      "headers":{  
         "Accept":"*/*",
         "CloudFront-Forwarded-Proto":"https",
         "CloudFront-Is-Desktop-Viewer":"true",
         "CloudFront-Is-Mobile-Viewer":"false",
         "CloudFront-Is-SmartTV-Viewer":"false",
         "CloudFront-Is-Tablet-Viewer":"false",
         "CloudFront-Viewer-Country":"AU",
         "Host":"980urz0ll1.execute-api.us-east-1.amazonaws.com",
         "User-Agent":"curl/7.54.0",
         "Via":"1.1 f27473d71ae79f24b43c80017fa9f378.cloudfront.net (CloudFront)",
         "X-Amz-Cf-Id":"KnEYELVFlb-u_362icYLObkGaglzBp5tZXiirGZSGSxeKsQiqhC1lA==",
         "X-Amzn-Trace-Id":"Root=1-59943ae0-610ab9ad71ecc20802c4d83f",
         "X-Forwarded-For":"x.x.x.x, x.x.x.x",
         "X-Forwarded-Port":"443",
         "X-Forwarded-Proto":"https"
      },
      "queryStringParameters":null,
      "pathParameters":null,
      "stageVariables":null,
      "requestContext":{  
         "path":"/dev/todos",
         "accountId":"905970388232",
         "resourceId":"hajs5j",
         "stage":"dev",
         "requestId":"ad792fa7-827e-11e7-8be4-d933f65d0e1d",
         "identity":{  
            "cognitoIdentityPoolId":null,
            "accountId":null,
            "cognitoIdentityId":null,
            "caller":null,
            "apiKey":"",
            "sourceIp":"175.34.244.76",
            "accessKey":null,
            "cognitoAuthenticationType":null,
            "cognitoAuthenticationProvider":null,
            "userArn":null,
            "userAgent":"curl/7.54.0",
            "user":null
         },
         "resourcePath":"/todos",
         "httpMethod":"POST",
         "apiId":"980urz0ll1"
      },
      "body":null,
      "isBase64Encoded":false
   }
}
```
Where did all this come from? If you remember above the "input" property on the json comes from the event parameter passed to our create javascript function. This time, our event was an http request rather than a simple command line request, so it comes with a whole bunch for extra information, which can be very useful.
Usually, event.body is what we're most interested in.

Let's start building our functioning API. 

We're going to need somewhere to store our Todos and for this we're going to use DynamoDB. We define the table we want to use and what permissions our service needs to have in `serverless.yml`.
Update the providers section of `serverless.yml` to look like this:
```yaml
provider:
  name: aws
  runtime: nodejs6.10
  environment:
    DB_TABLE: ${self:service}-${opt:stage, self:provider.stage}
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource: "arn:aws:dynamodb:${opt:region, self:provider.region}:*:table/${self:provider.environment.DB_TABLE}"
```
Let's take a look at what we're doing here. The new `enviroment` property allows to provide enviroment variables to our functions. When they're definied in the providers section like this they will be available to all functions within the service. So all of our functions will have access to the `DB_TABLE` enviroment variable, which is the service name + the stage we're using, in this case todomvc-api-dev. You can also define enviroment variables at the function level using the same syntax.

The `iamRoleStatments` section defines an IAM role, which is similar to a user in that it defines what an entity can and can't do with a resource. We're granting the role permission to query, scan, get, put, update and delete from our dynamoDB table, which is specified in the resource property.

We have our permissions set up for the table but we still need to actually create it. For this we'll be adding a new `resources` section to our `serverless.yml` file.
The `resources` section is made up of raw [CloudFormation](https://aws.amazon.com/cloudformation/) configuration. CloudFormation is Amazon's way of allow developers to create and manage resources through code. Add the following to the end of your `serverless.yml` file:
```yaml
resources:
  Resources:
    TodosDynamoDbTable:
      Type: 'AWS::DynamoDB::Table'
      DeletionPolicy: Retain
      Properties:
        AttributeDefinitions:
          -
            AttributeName: id
            AttributeType: S
        KeySchema:
          -
            AttributeName: id
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:provider.environment.DB_TABLE}
```
Here we've said we want to create a DynamoDB table with a key named id and a type of hash. We've provisioned the minimum read and write throughput which should be more than enough for what we're doing. 
The `DeletionPolicy` attribute defines what should happen when the stack that the database is part of is deleted. In this case we want the database to be persisted so we set it to retain.

Run `sls deploy` then login to your AWS Console, go to DynamoDB then tables and you should see a table called todomvc-api-dev.

We've now got all the infrastructure we need so let's start creating our todos!

If we look at the [API Spec](https://github.com/tastejs/todomvc-api/blob/master/todos.apib) to create a todo we should accept a json request with a title property and return a json object with an id, title and completed properties. In the spec the ID is a simple integer but we're going to use a UUID instead. This is mostly because of how DyanmoDB works. It's well outside the scope of this tutorial but you can read more about it [here](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GuidelinesForGSI.html#GuidelinesForGSI.UniformWorkloads).

We need to include the uuid and aws-sdk modules in our handler so open up `create.js` and add these lines to the top under 'use strict'.
```javascript
const uuid = require('uuid');
const AWS = require('aws-sdk'); // eslint-disable-line import/no-extraneous-dependencies

const dynamoDb = new AWS.DynamoDB.DocumentClient();
```
We need to handle the incoming json request. The data we want will be in the event body, so we need to parse that and check it is valid. Add the following to the start of your create function
```javascript
  const data = JSON.parse(event.body);
  if (typeof data.title !== 'string') {
    console.error('Validation Failed');
    callback(new Error('Couldn\'t create the todo item.'));
    return;
  }
```
Here we've passed the body then checked that it has a property title (typeof will return undefined if title does not exist) and that it is a string. If either of these fail, we log an error to the console and return an error over http using the callback.

After this we can be confident that we have a valid request so we can go ahead and generate an id for the todo, save it to the database and if all that was succesful, return a succesful response. Under the lines we just added, add
```javascript
    const params = {
        TableName: process.env.DB_TABLE,
        Item: {
            id: uuid.v1(),
            title: data.text,
            completed: false,
        },
    };

    // write the todo to the database
    dynamoDb.put(params, (error) => {
        // handle potential errors
        if (error) {
            console.error(error);
            callback(new Error('Couldn\'t create the todo item.'));
            return;
        }

        // create a response
        const response = {
            statusCode: 201,
            headers: {
                    "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
            },
            body: JSON.stringify(params.Item),
        };
        callback(null, response);
    });
```
`params` is an object containing the name of the table we want to insert into and the item we want to insert. We then pass this to the dynamoDb.put function which takes parameters as the first argument and a callback as the second. The callback is provide with an error argument which will contain erorr information if one occured and will be falsy if the insert was successful. 

You'll notice we're using [Arrow Functions](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Functions/Arrow_functions) to define our callback. Arrow functions are new in es6a and are effectively shorthand anonymous functions. They also don't bound `this` which can be very useful in object oriented projects.

Our code checks if an error occurs and if it did, returns an error to the users. If not, we return the succesfully inserted item. We also need to set the `Access-Control-Allow-Origin` header to ensure CORS requests work properly. Setting `cors` to true in `serverless.yml` enables support on the API gateway side, but our actual functions still need to return the header.

Your complete `create.js` file should now look like this:
```javascript
'use strict';
const uuid = require('uuid');
const AWS = require('aws-sdk'); // eslint-disable-line import/no-extraneous-dependencies

const dynamoDb = new AWS.DynamoDB.DocumentClient();

module.exports.create = (event, context, callback) => {
    const data = JSON.parse(event.body);
    if (typeof data.title !== 'string') {
        console.error('Validation Failed');
        callback(new Error('Couldn\'t create the todo item.'));
        return;
    }

    const params = {
        TableName: process.env.DB_TABLE,
        Item: {
            id: uuid.v1(),
            title: data.title,
            completed: false,
        },
    };

    // write the todo to the database
    dynamoDb.put(params, (error) => {
        // handle potential errors
        if (error) {
            console.error(error);
            callback(new Error('Couldn\'t create the todo item.'));
            return;
        }

        // create a response
        const response = {
            statusCode: 201,
            headers: {
                "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
            },
            body: JSON.stringify(params.Item),
        };
        callback(null, response);
    });
};
```

Let's deploy our function and try it out!
```shell
sls deploy
curl -H "Content-Type: application/json" -X POST -d '{"title":"My First Todo"}' https://yoururl/dev/todos
```
Hopefully that curl command returned something like this
```json
{"id":"2fbc57c0-82ea-11e7-8be7-697342addb6e","title":"My First Todo","completed":false}
```
If you look in the DynamoDB table via the AWS console you should be able to see the item has been inserted there too.

That's all we need to do for our create function! We're now well on our way to having a fully functioning CRUD API. Let's start create the other functions.

## List
The next function in the API spec is list. This function is straight forward - there is no request body to parse so we simply need to fetch the todos from the database and return them in the response. 
Create a new file `todos/list.js` and paste in the following code:
```javascript
'use strict';
const AWS = require('aws-sdk'); // eslint-disable-line import/no-extraneous-dependencies

const dynamoDb = new AWS.DynamoDB.DocumentClient();

module.exports.list = (event, context, callback) => {
    const params = {
        TableName: process.env.DB_TABLE,
    };

    // write the todo to the database
    dynamoDb.scan(params, (error, result) => {
        // handle potential errors
        if (error) {
            console.error(error);
            callback(new Error('Couldn\'t fetch the todo items.'));
            return;
        }

        // create a response
        const response = {
            statusCode: 200,
            headers: {
                "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
            },
            body: JSON.stringify(result.Items),
        };
        callback(null, response);
    });
};
```
Hopefully a lot of it looks familiar. The new piece here is the dynamoDb.scan function. The scan function always gets every item in the table. You can provide a filter expression but it is only applied after all items have been fetched. Using scan is fine and convinient for our use case but generally you should try to avoid it, see the [guidelines](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/QueryAndScanGuidelines.html).
Scan provides an error and a result argument to the callback. Again we check for an error and if none occured then we return the array of items we got from scan.

We've create our javascript function but we still need to tell serverless about it. Add the follow to the functions section of `serverless.yml`
```yaml
  list:
    handler: todos/list.list
    events:
      - http:
          path: todos
          method: get
          cors: true
```
This defines our new list lambda function, tells serverless requests are to be handle by the `list` function in our `todos/list.js` file and that it responds to `GET` requests on the todos path. This is the same path that our `create` function works on but that function only handled `POST` requests.

Let's deploy our new function
```shell
sls deploy
```
Hopefully you'll see a response which contains something like this:
```
endpoints:
  POST - https://980urz0ll1.execute-api.us-east-1.amazonaws.com/dev/todos
  GET - https://980urz0ll1.execute-api.us-east-1.amazonaws.com/dev/todos
```
Let's create a couple more todos so we can be sure our list function is working
```shell
curl -H "Content-Type: application/json" -X POST -d '{"title":"Buy milk"}' https://yoururl/dev/todos
curl -H "Content-Type: application/json" -X POST -d '{"title":"Re-evaluate life"}' https://yoururl/dev/todos
```
Now we can test our list function. Running
```shell
curl https://yoururl/dev/todos
``` 
Should return something like
```json
[{"id":"0a128530-82ef-11e7-ac45-1f2a09ee93e6","completed":false,"title":"Re-evaluate life"},{"id":"fc045860-82ee-11e7-ac45-1f2a09ee93e6","completed":false,"title":"Buy milk"},{"id":"2fbc57c0-82ea-11e7-8be7-697342addb6e","completed":false,"title":"My First Todo"}]
```

## Get
We can get the full list of todos but we don't have any way of getting a single todo yet. In your `serverless.yml` file add the follow to the functions section:
```yaml
  get:
    handler: todos/get.get
    events:
      - http:
          path: todos/{id}
          method: get
          cors: true
```
The only thing we haven't seen before here is `path: todos/{id}`. The curly braces allow to capture the id as a path parameter and pass it to our function.

Create a new file `todos/get.js` and paste in the following code:
```javascript
'use strict';
const AWS = require('aws-sdk'); // eslint-disable-line import/no-extraneous-dependencies

const dynamoDb = new AWS.DynamoDB.DocumentClient();

module.exports.get = (event, context, callback) => {
    const params = {
        TableName: process.env.DB_TABLE,
        Key: {
            id: event.pathParameters.id,
        },
    };

    // fetch the todo from the database
    dynamoDb.get(params, (error, result) => {
        // handle potential errors
        if (error) {
            console.error(error);
            callback(new Error('Couldn\'t fetch the todo item.'));
            return;
        }

        // create a response
        const response = {
            statusCode: 200,
            headers: {
                "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
            },
            body: JSON.stringify(result.Item),
        };
        callback(null, response);
    });
};
```
Here we're introduced to the dynamoDb.get function. Again this takes a parameters object as it's first argument. This object must contain a property `Key` which is an object with properties to match the key schema of the table. In our case the key schema is a single value, id.

Let's deploy our new function
```shell
sls deploy
```

First let's see what happens if we try to get a todo using an invalid id.
```shell
curl https://yoururl/dev/todos/1a
```
Hmm it didn't return anything at all, isn't that a bit strange?
To find out what's happening we'll use another one of serverless' handy features - log retrieval.
Add the follow two lines at the start of the dynamoDb.get callback
```javascript
console.log(error);
console.log(result);
```
Then run 
```shell
sls deploy
curl https://yoururl/dev/todos/1a
```
If you've ever tried to read the logs for a lambda function in CloudWatch you'll know that it's very messy. Serverless provides a command to make things much easier. Let's have a look at the logs for our get function by running
```shell
sls logs --function get
```
There will be some information about the requests we've executed but you should also see something like
```
2017-08-17 13:11:20.873 (+10:00)	bdc8cba6-82f9-11e7-930b-1d31bc7f52b1	null
2017-08-17 13:11:20.874 (+10:00)	bdc8cba6-82f9-11e7-930b-1d31bc7f52b1	{}
```
Which are the lines we logged to the console. We can see that error is null but no results were returned either. Getting no results isn't considered an error, so our function ends up returning nothing. Let's change our code to check we returned an item and if not return a 404 error.

```javascript
        // create a response
        let response = {}
        if(result.hasOwnProperty('Item')) {
            response = {
                statusCode: 200,
                headers: {
                    "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
                },
                body: JSON.stringify(result.Item),
            };
        } else {
            response = {
                statusCode: 404,
                headers: {
                    "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
                },
                body: JSON.stringify({'error':'Todo not found'}),
            };
        }
        callback(null, response);
```
Now we deploy again
```shell
sls deploy
curl https://yoururl/dev/todos/1a
```
and we get the response
```json
{"error":"Todo not found"}
```
Let's also check that it works for a valid todo id. Go back and get an id of one of the todos you created (you can see the list again by running `curl https://yoururl/dev/todos`) then run
```shell
curl https://yoururl/dev/todos/validId
```
where validId is the uuid of one of your todos. You should get a response like this
```json
{"id":"fc045860-82ee-11e7-ac45-1f2a09ee93e6","completed":false,"title":"Buy milk"}
```

## Update 
Now we need a way to mark our todos as completed, so we'll implement the update function.

Add the following to the functions section of `serverless.yml`
```yaml
  update:
    handler: todos/update.update
    events:
      - http:
          path: todos/{id}
          method: put
          cors: true
```
Again we're reusing a path and passing a path parameter, this time handling the `PUT` request type.

Create the file `todos/update.js` and add the following code
```javascript
'use strict';
const AWS = require('aws-sdk'); // eslint-disable-line import/no-extraneous-dependencies

const dynamoDb = new AWS.DynamoDB.DocumentClient();

module.exports.update = (event, context, callback) => {
    const data = JSON.parse(event.body);
    if(typeof data.title !== "string" || typeof data.completed !== 'boolean'){
        console.error('Validation Failed');
        callback(new Error('Couldn\'t update the todo item.'));
        return;
    }

    const params = {
        TableName: process.env.DB_TABLE,
        Key: {
            id: event.pathParameters.id,
        },
        UpdateExpression: "set title=:t, completed=:c",
        ExpressionAttributeValues:{
            ":t": data.title,
            ":c": data.completed,
        },
        ReturnValues:"ALL_NEW"
    };

    // update the todo in the database
    dynamoDb.update(params, (error, result) => {
        // handle potential errors
        if (error) {
            console.error(error);
            callback(new Error('Couldn\'t update the todo item.'));
            return;
        }

        // create a response
        let response = {}
        if(result.hasOwnProperty('Attributes')) {
            response = {
                statusCode: 200,
                headers: {
                    "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
                },
                body: JSON.stringify(result.Attributes),
            };
        } else {
            response = {
                statusCode: 404,
                headers: {
                    "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
                },
                body: JSON.stringify({'error':'Todo not found'}),
            };
        }
        callback(null, response);
    });
};
```

The most confusing part of this is `params` so let's have a closer look.
```javascript
   const params = {
        TableName: process.env.DB_TABLE,
        Key: {
            id: event.pathParameters.id,
        },
        UpdateExpression: "set title=:t, completed=:c",
        ExpressionAttributeValues:{
            ":t": data.title,
            ":c": data.completed,
        },
        ReturnValues:"ALL_NEW"
    };
```
The first part if familiar. We're saying which table to use and matching the item with the id provided in the url path.
`UpdateExpression` is a string defining which attributes to update. You can also add new attributes this way. 
`ExpressionAtrributeValues` contains the values to to be substituted for the placeholders in `UpdateExpression.`
`ReturnValues` defines which attributes that should be returned. Here we saw to return all attributes of the updated item, rather than `UPDATED_NEW` which would only return the updated attributes.

Let's deploy our function then try marking a todo as completed
```shell 
sls deploy
curl -H "Content-Type: application/json" -X PUT \
-d '{"title":"Buy Milk", "completed":"true"}' \
https://yoururl/dev/todos/validId
```
You should get a response like
```json
{
    "completed": true,
    "id": "fc045860-82ee-11e7-ac45-1f2a09ee93e6",
    "title": "Buy Milk"
}
```

## Delete
We have create, read and update methods so now we just need to add a delete method to complete our CRUD API.

Add the follow to your `serverless.yml`
```yaml
  delete:
    handler: todos/delete.delete
    events:
      - http:
          path: todos/{id}
          method: delete
          cors: true
```

Now create `todos/delete.js` and paste in the following code:
```javascript
'use strict';
const AWS = require('aws-sdk'); // eslint-disable-line import/no-extraneous-dependencies

const dynamoDb = new AWS.DynamoDB.DocumentClient();

module.exports.delete = (event, context, callback) => {
    const params = {
        TableName: process.env.DB_TABLE,
        Key: {
            id: event.pathParameters.id,
        },
    };

    // delete the todo from the database
    dynamoDb.delete(params, (error) => {
        // handle potential errors
        if (error) {
            console.error(error);
            callback(new Error('Couldn\'t delete the todo item.'));
            return;
        }

        // create a response
        const response = {
                statusCode: 204,
                headers: {
                    "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
                },
        };
        callback(null, response);
    });
};
```
The dynamoDb.delete functions nearly identically to the get function, so the code should be easy to understand. A response with a 204 status code can not have a body so there's no need to specify one.

Let's deploy it and test it out
```shell
sls deploy
curl -X DELETE https://yoururl/dev/todos/validId
curl https://yoururl/dev/todos/
```
The todo you deleted should no longer appear in the list.

## Archive
There's one more function specified in the API Spec and that is the archive function. This function should delete all todos which have their completed property set to true.

Add the following to the functions section of your `serverless.yml`
```yaml
  archive:
    handler: todos/archive.archive
    events:
      - http:
          path: todos
          method: delete
          cors: true
```
This ensures that delete requests to /todos path (rather than /todos/{id}) will be handled by our archive function.

Create the files `todos/archive.js` and paste in the following code:
```javascript
'use strict';
const AWS = require('aws-sdk'); // eslint-disable-line import/no-extraneous-dependencies

const dynamoDb = new AWS.DynamoDB.DocumentClient();

module.exports.archive = (event, context, callback) => {
    const params = {
        TableName: process.env.DB_TABLE,
        FilterExpression: "completed = :val",
        ExpressionAttributeValues: {
            ":val": true
        },
    };

    // fetch all the todos from the database
    dynamoDb.scan(params, (error, result) => {
        // handle potential errors
        if (error) {
            console.error(error);
            callback(new Error('Couldn\'t fetch the todo items.'));
            return;
        }

        //Loop through and delete the returned items.
        let processed = 0;
        result.Items.forEach(item => {
            const deleteParams = {
                TableName: process.env.DB_TABLE,
                Key: {
                    id: item.id,
                },
            };
            dynamoDb.delete(deleteParams, (error) => {
                // handle potential errors
                if (error) {
                    console.log(item);
                    console.error(error);
                    callback(new Error('Couldn\'t delete the todo item.'));
                    return;
                }

                //Increase the count of processed items, then check if we're done
                processed++;
                if(processed == result.Items.length) {
                    // create a response
                    const response = {
                        statusCode: 204,
                        headers: {
                            "Access-Control-Allow-Origin" : "*" // Required for CORS support to work
                        },
                    };
                    callback(null, response);
                }
            });
        });
    });
};
```
There's a couple of new things here, so let's go through them.

We've used dynamoDb.scan before but this time we're putting a filter on it
```javascript
    const params = {
        TableName: process.env.DB_TABLE,
        FilterExpression: "completed = :val",
        ExpressionAttributeValues: {
            ":val": true
        },
    };
```
The combination of `FilterExpression` and `ExpressionAttributeValues` ensure that only items with their completed attribute set to true will be returned by the scan. Note that, as has already been mentioned, the scan function actually still gets every item in the table before applying this filter, so it still uses the same amount of read capacity. 

Once the items have been returned, we want to delete them. Dynamodb does not have a bulk delete function, so we need for call the delete function on each item individually, hence this part of the code
```javascript
        //Loop through and delete the returned items.
        let processed = 0;
        result.Items.forEach(item => {
            const deleteParams = {
                TableName: process.env.DB_TABLE,
                Key: {
                    id: item.id,
                },
            };
            dynamoDb.delete(deleteParams, (error) => {
```
You'll notice the `let processed = 0;` line. This is needed because the dynamoDb functions are all asynchronous - we don't wait for one to completed before moving on to the next item in the loop and firing off its delete function.
The deletes won't necessarily complete in order but we want to be sure they have all completed before we return a request to the user. So on each loop we increase the processed count by one. If the processed count is now the same as the length of the Items array, we can be sure all items have been deleted and return a response. 
```javascript
                //Increase the count of processed items, then check if we're done
                processed++;
                if(processed == result.Items.length) {
                    // create a response
                    const response = {
                        statusCode: 204,
                    };
                    callback(null, response);
                }
```

Deploy your function and test it out
```shell
sls deploy
curl -X DELETE https://yoururl/dev/todos/
curl https://yoururl/dev/todos/
```
There shouldn't be any todos with complete = true returned by the second curl request. You might want to create a few more todos and changed their completed status to play around.

## Hooking it up to the frontend
We're now ready to hook up our API to the frontend. For convenience, there's a hosted version of the [Todo Client](https://itmasters.github.io/serverless-todomvc-client/) where you can copy and paste in your root url (https://yoururl/dev/todos), hit enter and everything should be good to go.

You can also run the same site locally 
```shell
git clone https://github.com/ITMasters/serverless-todomvc-client.git
cd serverless-todomvc-client
npm install
npm run start
```
To skip entering your url modify the file `/src/app/components/todo/todo.service.ts`
And change line 25 from
```javascript
public apiUrl = '';  // URL to web api
``` 
to 
```javascript
public apiUrl = 'https://yoururl/dev/todos';  // URL to web api
```

## Wrapping up
We've covered a lot - how to deploy a function to AWS Lambda, hook to up to a http endpoint, provisioning and using dynamoDB to create, get, update and retrieve records, and how to view logs from your functions. 

This was a simple application but will hopefully give you the building blocks you need to develop fully fledged serverless applications in the future!

Cheers,

Hamish
