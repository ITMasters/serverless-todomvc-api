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
npm install --save aws-sdk uuid
```

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
            body: JSON.stringify(params.Item),
        };
        callback(null, response);
    });
```
`params` is an object containing the name of the table we want to insert into and the item we want to insert. We then pass this to the dynamoDb.put function which takes parameters as the first argument and a callback as the second. The callback is provide with an error argument which will contain erorr information if one occured and will be falsy if the insert was successful. 

You'll notice we're using [Arrow Functions](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Functions/Arrow_functions) to define our callback. Arrow functions are new in es6a and are effectively shorthand anonymous functions. They also don't bound `this` which can be very useful in object oriented projects.

Our code checks if an error occurs and if it did, returns an error to the users. If not, we return the succesfully inserted item.

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

