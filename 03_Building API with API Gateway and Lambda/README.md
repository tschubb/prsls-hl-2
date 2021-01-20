# Module 3: Building API with API Gateway and Lambda

## Returning HTML from Lambda

**Goal:** Create and deploy an API that returns HTML instead of JSON

<details>
<summary><b>Create a serverless project</b></summary><p>

1. Create a directory for your serverless project.

    ```
    mkdir production-ready-serverless-workshop
    cd production-ready-serverless-workshop
    ```

2. Initialise the project:
    
    `npm init -y`
    
3. Install the `Serverless` framework as dev dependency.

    `npm install --save-dev serverless`

    Add `sls` to npm scripts by editing your `package.json` so your `scripts` section looks like this:

    ```json
      "scripts": {
        "sls": "serverless"
      },
    ```

4. Create nodejs Serverless project using one of the default templates.

    `npx sls create --template aws-nodejs`

    See more information about `serverless create` command on [CLI documentation](https://serverless.com/framework/docs/providers/aws/cli-reference/create/) page.

5. Delete the `handler.js` file at the root

</p></details>

<details>
<summary><b>Add an API function</b></summary><p>

1. Modify the `serverless.yml` to the following.

```yml
service: workshop-${self:custom.name}

custom:
  name: <YOUR_NAME_HERE>
  email: <YOUR_EMAIL_HERE>

provider:
  name: aws
  runtime: nodejs12.x

functions:
  get-index:
    handler: functions/get-index.handler
    events:
      - http:
          path: /
          method: get
```

2. Under `custom`, replace `<YOUR_NAME_HERE>` with your name (no whitespaces, all lower case), e.g. `yancui`. This is so that if your project doesn't clash with your colleague's in case you're using a shared AWS account.

3. Also, replace `<YOUR_EMAIL_HERE>` with your email, we'll use this later to set up email notification when someone places an order.

4. In the project root, add a folder called `functions`

3. Add a file `get-index.js` under the newly created `functions` folder

4. Modify the `get-index.js` file to the following:

```javascript
const fs = require("fs")

const html = fs.readFileSync('static/index.html', 'utf-8')

module.exports.handler = async (event, context) => {
  const response = {
    statusCode: 200,
    headers: {
      'Content-Type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

Here, we're loading a static HTML page from a `static` folder (to be added in the next section). Which we will return in the HTTP response, along with a header with the correct `Content-Type` (otherwise it defaults to `application/json`).

And since the variable `html` is declared and assigned OUTSIDE the handler function, it will run ONLY the first time our code executes on a new container. The same goes to any variables you declare outside the handler function, such as the `fs` module which we required at the top.

</p></details>

<details>
<summary><b>Add the static HTML</b></summary><p>

1. Add a folder to the project root, call it `static`

2. Add a file `index.html` under the newly created `static` folder

3. Modify the `index.html` file to the following:

```xml
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>
    
    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;
      }

      .column-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: -webkit-box;
        display: -moz-box;
        display: -ms-flexbox;
        display: -webkit-flex;
        display: flex;
        flex-flow: column;
        justify-content: center;
      }

      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        justify-content: center;
      }

      input {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }

      button {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
    </style>

    <script>
    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="search()">Find Restaurants</button>
        </li>
      </ul>
  </div>
  </body>

</html>
```

</p></details>

<details>
<summary><b>Deploy the serverless project</b></summary><p>

1. Run `deploy` command:

    `npx sls deploy`

    See more information about `deploy` command on [CLI documentation](https://serverless.com/framework/docs/providers/aws/cli-reference/deploy/) page.

2. In the output you should see something like this:

```
endpoints:
  GET - https://xxxxx.execute-api.us-east-1.amazonaws.com/dev/
```

Load the endpoint in the browser and see something like the following:

![](/images/mod03-001.png)

</p></details>

## Creating the restaurants API

**Goal:** Create and deploy the `/restaurants` endpoint

<details>
<summary><b>Add a function to return restaurants</b></summary><p>

1. Add a file `get-restaurants.js` file to the `functions` folder

2. Install the `aws-sdk` package

`npm install --save-dev aws-sdk`

3. Modify `get-restaurants.js` to the following:

```javascript
const DocumentClient = require('aws-sdk/clients/dynamodb').DocumentClient
const dynamodb = new DocumentClient()

const defaultResults = process.env.defaultResults || 8
const tableName = process.env.restaurants_table

const getRestaurants = async (count) => {
  console.log(`fetching ${count} restaurants from ${tableName}...`)
  const req = {
    TableName: tableName,
    Limit: count
  }

  const resp = await dynamodb.scan(req).promise()
  console.log(`found ${resp.Items.length} restaurants`)
  return resp.Items
}

module.exports.handler = async (event, context) => {  
  const restaurants = await getRestaurants(defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
}
```

This function depends on two environment variables:

* `defaultResults` [optional] : how many restaurants to return

* `restaurants_table` [required] : name of the restaurants DynamoDB table

Also notice that we didn't require the full `aws-sdk` at the top. This is because requiring only the client we need (in this case, the `DynamoDB.DocumentClient` client) has a big impact on cold start performance. See this [blog post](https://theburningmonk.com/2019/03/just-how-expensive-is-the-full-aws-sdk/) for more details.

4. Modify `serverless.yml` to add the new function and its environment variables:

```yml
get-restaurants:
  handler: functions/get-restaurants.handler
  events:
    - http:
        path: /restaurants
        method: get
  environment:
    restaurants_table: !Ref RestaurantsTable
```

Notice that the `restaurants_table` environment variable is referencing (using the CloudFormation pseudo function `!Ref`) a resource called `RestaurantsTable`. We'll add that next.

**IMPORTANT**: make sure this block is indented and aligns with the `get-index` function. e.g.

```yml
functions:
  get-index:
    handler: functions/get-index.handler
    events:
      - http:
          path: /
          method: get

  get-restaurants:
    handler: functions/get-restaurants.handler
    events:
      - http:
          path: /restaurants/
          method: get
    environment:
      restaurants_table: !Ref RestaurantsTable
```

</p></details>

<details>
<summary><b>Add a DynamoDB table in the serverless.yml</b></summary><p>

1. Add DynamoDB table to `serverless.yml`. Add this block to the end of the file:

```yml
resources:
  Resources:
    RestaurantsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: name
            AttributeType: S
        KeySchema:
          - AttributeName: name
            KeyType: HASH
```

This is how we define a DynamoDB table in CloudFormation. Couple of things to note if you're new to AWS or CloudFormation.

`RestaurantsTable` is called the **logical Id** in CloudFormation, and is the unique ID for a resource within a CloudFormation template. Other resources can reference this resource using this logical id. You typically use either the `!Ref` or `!GetAtt` function to reference a resource and get its attributes (like name or ARN). Which, depending on the resource and the attribute you want to get, you have to choose the right function... I have a cheatsheet [here](https://theburningmonk.com/cloudformation-ref-and-getatt-cheatsheet) that list all the resources and what you get with each.

DynamoDB two billing modes. And here, we're using the `On-Demand` mode by setting `BillingMode` to `PAY_PER_REQUEST`. This means we don't pay for the table unless we use it, which is perfect for a demo app like this. In fact, this is probably the right setting for your app in production too unless you have very stable and very predictable load.

And lastly, DynamoDB operates with `HASH` and `RANGE` keys as the only schemas you need to specify, so that's what `KeySchema` is doing. When you specify an attribute in the key schema you also need to add them to the `AttributeDefinitions` list so you can specify their type (`S` for `String`), this is how we tell DynamoDB that the table has a `HASH` key called `name` and it's a `S`tring.

There are other configurations available, but they're needed here. For more details, check out the CloudFormation docs [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html).

Ok, now let's deploy the serverless project again (so CloudFormation provisions the table):

`npx sls deploy`

After deployment finishes, the DynamoDB table would now be provisioned.

If you look into the DynamoDB console in your AWS account, you will see that the table would have an auto-generated name like `workshop-yancui-dev-RestaurantsTable-1867PM9U2N4ZA`.

At this point, it's also worth peeking into the CloudFormation console, and see that the Serverless framework generated a CloudFormation stack for our project, and the name of the stack is derived from the `service` and `stage` name of our project - e.g. `workshop-yancui-dev`.

The stack includes a no. of outputs including the deployment bucket name, and the root URL for API Gateway REST API.

![](/images/mod03-002.png)

Since we have added a DynamoDB table to the stack as a custom resource, it's useful to add its name to the stack output.

2. To add the `RestaurantsTable` name to stack output, go to `serverless.yml`. Under the `resources` section, add the following:

```yml
  Outputs:
    RestaurantsTableName:
      Value: !Ref RestaurantsTable
```

**IMPORTANT**: `Outputs` should be aligned with `Resources` (**NOT** `resources`). So after this change, the `resources` section of your `serverless.yml` should look like this:

```yml
resources:
  Resources:
    RestaurantsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: name
            AttributeType: S
        KeySchema:
          - AttributeName: name
            KeyType: HASH

  Outputs:
    RestaurantsTableName:
      Value: !Ref RestaurantsTable
```

Make sure the indentations are exactly the same.

3. Deploy the project again

`npx sls deploy`

and check in the CloudFormation console to make sure that you see the `RestaurantsTableName` in the stack outputs

![](/images/mod03-003.png)

</p></details>

<details>
<summary><b>Seeding data into the restaurant table</b></summary><p>

To seed some restaurant data into the table, we're going to add a script to do that. 

But wait! We have a problem here...

The DynamoDB table name is randomly generated, how do we write this script so that it doesn't require us to go into the AWS console and copy the name of the table?

There are a couple of solutions to this:

1. Use the [serverless-export-env](https://github.com/arabold/serverless-export-env/) plugin to export the environment variables to a `.env` file.
2. Programmatically reference the `RestaurantsTableName` stack output we added above.
3. Export the table name as a SSM parameter (by adding a `AWS::SSM::Parameter` resource to our `serverless.yml`) and programmatically reference it from the script.

For the purpose of this demo app, let's use option 1, as it'll come in handy for us later on when we start writing tests. See [Paul Swail](https://twitter.com/paulswail)'s [post](https://winterwindsoftware.com/cloud-config-local-tests/) for more details.

1. Install the `serverless-export-env` plugin by running

`npm install --save-dev serverless-export-env`

2. Open `serverless.yml` and add the following to the end of the file

```yml
plugins:
  - serverless-export-env
```

**IMPORTANT**: `plugins` should be at the same level as `provider`, `functions` and `resources`, that is, it should have *NO INDENTATION*.

3. Run `npx sls export-env`. This command should generate a `.env` file in your project root, and the file content should look something like this:

`restaurants_table=workshop-yancui-dev-RestaurantsTable-1Y097GF7QLUIX`

This is because our `get-restaurants` function has an environment variable called `restaurants_table`. The `serverless-export-env` plugin has resolved the `!Ref RestaurantTable` reference and helpfully added the resolved table name to the `.env` file.

To load this `.env` file from our code, we need the `dotenv` NPM package.

4. Add `.env` to your `.gitignore` file

5. Install the `dotenv` package

`npm install --save-dev dotenv`

6. Add a file `seed-restaurants.js` to the project root

7. Modify `seed-restaurants.js` to the following.

```javascript
const AWS = require('aws-sdk')
AWS.config.region = 'us-east-1'
const dynamodb = new AWS.DynamoDB.DocumentClient()
require('dotenv').config()

const restaurants = [
  { 
    name: "Fangtasia", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/fangtasia.png", 
    themes: ["true blood"] 
  },
  { 
    name: "Shoney's", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/shoney's.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Freddy's BBQ Joint", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/freddy's+bbq+joint.png", 
    themes: ["netflix", "house of cards"] 
  },
  { 
    name: "Pizza Planet", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/pizza+planet.png", 
    themes: ["netflix", "toy story"] 
  },
  { 
    name: "Leaky Cauldron", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/leaky+cauldron.png", 
    themes: ["movie", "harry potter"] 
  },
  { 
    name: "Lil' Bits", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/lil+bits.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Fancy Eats", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/fancy+eats.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Don Cuco", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/don%20cuco.png", 
    themes: ["cartoon", "rick and morty"] 
  },
];

const putReqs = restaurants.map(x => ({
  PutRequest: {
    Item: x
  }
}))

const req = {
  RequestItems: {
    [process.env.restaurants_table]: putReqs
  }
}
dynamodb.batchWrite(req).promise()
  .then(() => console.log("all done"))
  .catch(err => console.error(err))
```

8. Run the `seed-restaurants.js` script

`node seed-restaurants.js`

</p></details>

<details>
<summary><b>Configure IAM permission</b></summary><p>

1. Modify `serverless.yml` and add an `iamRoleStatements` section under `provider` (make sure you check for proper indentation!):

```yml
provider:
  name: aws
  runtime: nodejs12.x

  iamRoleStatements:
    - Effect: Allow
      Action: dynamodb:scan
      Resource: !GetAtt RestaurantsTable.Arn
```

This adds the `dynamodb:scan` permission to the Lambda execution role.

2. Deploy the project

`npx sls deploy`

</p></details>

## Displaying restaurants in the landing page

**Goal:** Displays the restaurants in `index.html`

<details>
<summary><b>Update index.html to allow templating with Mustache</b></summary><p>

1. Modify `index.html` to the following:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>
    
    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;
      }
      .restaurantsDiv {
        background-color: #ffffff;
        width: 100%;
        height: auto;
      }
      .dayOfWeek {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 32px;
        padding: 10px;
        height: auto;
        display: flex;
        justify-content: center;
      }
      .column-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: column;
        flex-wrap: wrap;
        justify-content: center;
      }
      .row-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .restaurant {
        background-color: #00a8f7;
        border-radius: 10px;
        padding: 5px;
        height: auto;
        width: auto;
        margin-left: 40px;
        margin-right: 40px;
        margin-top: 15px;
        margin-bottom: 0px;
        display: flex;
        justify-content: center;
      }
      .restaurant-name {
        font-size: 24px;
        font-family:Arial, Helvetica, sans-serif;
        color: #ffffff;
        padding: 10px;
        margin: 0px;
      }
      .restaurant-image {
        padding-top: 0px;
        margin-top: 0px;
      }
      input {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
      button {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
    </style>

    <script>
    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="search()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul class="row-container">
              {{#restaurants}}
              <li class="restaurant">
                <ul class="column-container">
                    <li class="item restaurant-name">{{name}}</li>
                    <li class="item restaurant-image">
                      <img src="{{image}}">
                    </li>
                </ul>
              </li>
              {{/restaurants}}
            </ul> 
          </div>
        </li>
      </ul>
  </div>
  </body>

</html>
```
</p></details>

<details>
<summary><b>Render the index.html with restaurants</b></summary><p>

1. Install `mustache` as dependency

`npm install --save mustache`

2. Install `axios` as dependency

`npm install --save axios`

3. Modify `get-index.js` to the following:

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('axios')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']

let html

function loadHtml () {
  if (!html) {
    console.log('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    console.log('loaded')
  }
  
  return html
}

const getRestaurants = async () => {
  const httpReq = http.get(restaurantsApiRoot)
  return (await httpReq).data
}

module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const html = Mustache.render(template, { dayOfWeek, restaurants })
  const response = {
    statusCode: 200,
    headers: {
      'Content-Type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

After this change, the `get-index` function needs the `restaurants_api` environment variable to know where the `/restaurants` endpoint is.

The good news is that, the Serverless always use the logical ID `ApiGatewayRestApi` for the API Gateway REST API resource. So you can easily construct the URL for the `/restaurants` endpoint.

The bad news is that you end up writing something like:

```yml
Fn::Join:
  - ''
  - - "https://"
    - !Ref ApiGatewayRestApi
    - ".execute-api.${self:provider.region}.amazonaws.com/${self:provider.stage}/restaurants"
```

it works, but is pretty verbose!

Luckily for us, there's a plugin that can help us with that. Enter `serverless-pseudo-parameters`, which allows us to use CloudFormation pseudo parameters such as `AWS::AccountId` and `AWS::Region`, and reference other resources easily.

4. Install the `serverless-pseudo-parameters` plugin

`npm install --save-dev serverless-pseudo-parameters`

5. Add `serverless-pseudo-parameters` to the list of `plugins` at the bottom of the `serverless.yml` file. Your `plugins` section should look like this afterwards

```yml
plugins:
  - serverless-export-env
  - serverless-pseudo-parameters
```

6. Modify the `serverless.yml` to add the `restaurants_api` environment variable to the `get-index` function:

```yml
get-index:
  handler: functions/get-index.handler
  events:
    - http:
        path: /
        method: get
  environment:
    restaurants_api: https://#{ApiGatewayRestApi}.execute-api.#{AWS::Region}.amazonaws.com/${self:provider.stage}/restaurants
```

Notice the `#{...}` values in that string. This is the syntax that the `serverless-pseudo-parameters` plugin introduces. Under the hood, the plugin translates strings that contain `#{...}` into a `Fn::Sub`, which you can see in the generated CloudFormation update template:

```JSON
"Environment": {
  "Variables": {
    "restaurants_api": {
      "Fn::Sub": "https://${ApiGatewayRestApi}.execute-api.${AWS::Region}.amazonaws.com/dev/restaurants"
    }
  }
}
```

* `#{ApiGatewayRestApi}` => `${ApiGatewayRestApi}`
* `#{AWS::Region}` => `${AWS::Region}`, which unfortunately you can't use within the `serverless.yml` as the Serverless framework also uses `${...}` in its own variables system. See [this page](https://serverless.com/framework/docs/providers/aws/guide/variables/) for more information.

7. Deploy the project

`npx sls deploy`

Reload the `index.html` and you should see something like the following

![](/images/mod03-004.png)

</p></details>
