# Module 23: X-Ray

Invocation traces in X-Ray

<details>
<summary><b>Integrating with X-Ray</b></summary><p>

1. In the `serverless.yml` under the `provider` section, add the following:

```yml
tracing:
  apiGateway: true
  lambda: true
```

**NOTE** this should align with `name`, `runtime` and `environment`.

2. Add the following back to the `provider` section:

```yml
iamRoleStatements:
  - Effect: Allow
    Action:
      - "xray:PutTraceSegments"
      - "xray:PutTelemetryRecords"
    Resource: "*"
```

**IMPORTANT** this should be aligned with `provider.tracing` and `provider.environment`. e.g.

```yml
provider:
  name: aws
  runtime: nodejs12.x
  stage: dev
  environment:
    ...
  tracing:
    apiGateway: true
    lambda: true
  iamRoleStatements:
    - Effect: Allow
      Action:
        - "xray:PutTraceSegments"
        - "xray:PutTelemetryRecords"
      Resource: "*"
```

This enables X-Ray tracing for all the functions in this project. Normally, when you enable X-Ray tracing in the `provider.tracing` the Serverless framework would add these permissions for you automatically. However, since we're using the `serverless-iam-roles-per-function`, these additional permissions are not passed along...

So far, the best workaround I have found, short of fixing the plugin to do it automatically, is to add this blob back to the `provider` section and tell the plugin to inherit these shared permissions in each function's IAM role.

To do that, we need the functions to inherit the permissions from this default IAM role.

3. Modify `serverless.yml` to add the following to the `custom` section

```yml
serverless-iam-roles-per-function:
  defaultInherit: true
```

This is courtesy of the `serverless-iam-roles-per-function` plugin, and tells the per-function roles to inherit these common permissions.

4. Deploy the project

`npx sls deploy`

5. Load up the landing page, and place an order. Then head to the X-Ray console and see what you get.

![](/images/mod23-001.png)

![](/images/mod23-002.png)

![](/images/mod23-003.png)

As you can see, you get some useful bits of information. However, if I were to debug performance issues of, say, the `get-restaurants` function, I need to see how long the call to DynamoDB took, that's completely missing right now.

To make our traces more useful, we need to capture more information about what our functions are doing. To do that, we need more instrumentation.

</p></details>

<details>
<summary><b>Instrumenting AWSSDK</b></summary><p>

At the moment we're not getting a lot of value out of X-Ray. We can get much more information about what's happening in our code if we instrument the various steps.

To begin with, we can instrument the AWS SDK so we track how long calls to DynamoDB and SNS takes in the traces.

1. Install `aws-xray-sdk-core` as dependency

`npm install --save aws-xray-sdk-core`

2. Modify `functions/get-restaurants.js` and replace 

```js
const dynamodb = new DocumentClient()
``` 

with the following

```javascript
const dynamodb = new DocumentClient()
const XRay = require('aws-xray-sdk-core')
XRay.captureAWSClient(dynamodb.service)
```

This instruments the DynamoDB client, so that it will emit additional trace segments so you can see how long the DynamoDB `Scan` operation took in the `get-restaurants` function.

![](/images/mod23-004.png)

3. Repeat step 2 for `functions/search-restaurants.js`

4. Open `functions/notify-restaurant.js`

and replace 

```js
const eventBridge = new EventBridge()
```

with the following

```javascript
const XRay = require('aws-xray-sdk-core')
const eventBridge = XRay.captureAWSClient(new EventBridge())
```

and then replace 

```js
const sns = new SNS()
```

with the following

```js
const sns = XRay.captureAWSClient(new SNS())
```

This allows us to trace the calls to SNS and EventBridge in the `notify-restaurant` function.

![](/images/mod23-005.png)

5. Repeat step 4 for `functions/place-order.js` (minus the SNS step since it doesn't need the SNS client).

6. Deploy the project

`npx sls deploy`

7. Load up the landing page, and place an order. Then head to the X-Ray console and see what you get now.

If you look at a few of the traces for just the `get-restaurants` function, which you can do by going back to the traces view (and make sure the search box is empty).

Click the link for the `/restaurants` URL:

![](/images/mod23-006.png)

This should add the filter for the `/restaurants` path, and show you only the traces for the `get-restaurants` function.

![](/images/mod23-007.png)

If you open a few of these traces, you might notice that the DynamoDB requests take somewhere between 30-80ms. That's an awful long time considering that DynamoDB averages single-digit latency. Most of that time is setting up the HTTPs connection, which unfortunately, is not reused by default by the Node.js AWS SDK (soon to be set as default in v3)!

More details about this [here](https://theburningmonk.com/2019/02/lambda-optimization-tip-enable-http-keep-alive/).

8. Now, let's apply the **single most effective performance optimize** for a Node.js function :-)

Open `serverless.yml` and add the following environment variable to `provider.environment`:

```yml
AWS_NODEJS_CONNECTION_REUSE_ENABLED: 1
```

and redeploy

`npx sls deploy`

9. Reload the homepage a couple of times, and look at the traces for the `get-restaurants` function. Notice how much faster the subsequent invocations are! The effects are additive too, as every single request through the AWS SDK required HTTPs handshake...

</p></details>

<details>
<summary><b>Instrumenting HTTP calls</b></summary><p>

We can get even more value if we could see the traces for `get-index` function and the corresponding trace for the `get-restaurants` function in one screen.

![](/images/mod23-008.png)

Then it's proper distributed tracing! It's not very helpful if you're restricted to only what happens inside one function.

Fortunately, you can instrument the built-in `https` module with the X-Ray SDK, unfortunately, you have to use it instead of other HTTP clients..

1. Modify `functions/get-index.js` and add the following to the **top of the file**

```javascript
const AWSXRay = require('aws-xray-sdk-core')
AWSXRay.captureHTTPsGlobal(require('https'))
```

2. Deploy the project

`npx sls deploy`

3. Load up the landing page, and place an order. Then head to the X-Ray console and now you can see the traces for `get-index` and `get-restaurants` function in one place.

</p></details>

<details>
<summary><b>Fix broken tests</b></summary><p>

If you run the integration tests now

`npm run test`

then you'll see the some tests are failing...

This is because the X-Ray SDK expects some context and root segment to be provided by the Lambda service's runtime. Which we won't have when running locally.

To fix this, we need to tell the X-Ray SDK to not crash when these contexts are missing, and log an error instead.

1. Modify `steps/init.js` to add this along with other environment variables (where we set the `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_REGION` environment variables, somewhere near there):

```js
process.env.AWS_XRAY_CONTEXT_MISSING = 'LOG_ERROR'
```

This stops the X-Ray SDK from erroring when it doesn't find the context

Rerun the integration tests, and the tests are passing, but with a lot of error messages...

```
 PASS  tests/test_cases/notify-restaurant.tests.js
 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.error node_modules/aws-xray-sdk-core/lib/logger.js:19
      2020-05-18 18:05:03.480 +02:00 [ERROR] Error: Failed to get the current sub/segment from the context.
          at Object.contextMissingLogError [as contextMissing] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:26:19)
          at Object.getSegment (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:92:45)
          at Object.resolveSegment (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:73:19)
          at captureOutgoingHTTPs (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/patchers/http_p.js:97:31)
          at Object.captureHTTPsRequest [as request] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/patchers/http_p.js:185:12)
          at RedirectableRequest._performRequest (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/follow-redirects/index.js:169:24)
          at new RedirectableRequest (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/follow-redirects/index.js:66:8)
          at Object.wrappedProtocol.request (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/follow-redirects/index.js:307:14)
          at dispatchHttpRequest (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/axios/lib/adapters/http.js:179:25)
          at new Promise (<anonymous>)

 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.error node_modules/aws-xray-sdk-core/lib/logger.js:19
      2020-05-18 18:05:04.259 +02:00 [ERROR] Error: Failed to get the current sub/segment from the context.
          at Object.contextMissingLogError [as contextMissing] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:26:19)
          at Object.getSegment (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:92:45)
          at Object.resolveSegment (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:73:19)
          at features.constructor.captureAWSRequest [as customRequestHandler] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/patchers/aws_p.js:60:29)
          at features.constructor.addAllRequestListeners (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/service.js:283:12)
          at features.constructor.makeRequest (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/service.js:203:10)
          at features.constructor.svc.(anonymous function) [as scan] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/service.js:677:23)
          at DocumentClient.makeServiceRequest (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/dynamodb/document_client.js:97:42)
          at DocumentClient.scan (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/dynamodb/document_client.js:360:17)
          at getRestaurants (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/functions/get-restaurants.js:22:31)

 PASS  tests/test_cases/place-order.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info functions/search-restaurants.js:33
      this is a new secret
    console.error node_modules/aws-xray-sdk-core/lib/logger.js:19
      2020-05-18 18:05:05.715 +02:00 [ERROR] Error: Failed to get the current sub/segment from the context.
          at Object.contextMissingLogError [as contextMissing] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:26:19)
          at Object.getSegment (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:92:45)
          at Object.resolveSegment (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/context_utils.js:73:19)
          at features.constructor.captureAWSRequest [as customRequestHandler] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-xray-sdk-core/lib/patchers/aws_p.js:60:29)
          at features.constructor.addAllRequestListeners (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/service.js:283:12)
          at features.constructor.makeRequest (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/service.js:203:10)
          at features.constructor.svc.(anonymous function) [as scan] (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/service.js:677:23)
          at DocumentClient.makeServiceRequest (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/dynamodb/document_client.js:97:42)
          at DocumentClient.scan (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/node_modules/aws-sdk/lib/dynamodb/document_client.js:360:17)
          at findRestaurantsByTheme (/Users/yancui/SourceCode/workshops/prsls-online-may-2020-demo/functions/search-restaurants.js:25:31)


Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        4.24s
Ran all test suites.
```

2. One thing you could do, is to monkey-patch `console.error` with an anonymous mock function using `jest`. For example, in `steps/init.js`, and somewhere in the `init` function, add this one line:

```javascript
console.error = jest.fn()
```

3. Rerun the integration tests

`npm run test`

and see that all the tests should be passing now, and there're no a sea of error texts.

```
 PASS  tests/test_cases/notify-restaurant.tests.js
  ● Console

    console.debug node_modules/@dazn/lambda-powertools-logger/index.js:82
      {"message":"notified restaurant","orderId":"510dba15-69bd-582e-a82b-7caa168f32ae","restaurantName":"Fangtasia","awsRegion":"us-east-1","debug-log-enabled":"true","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
    console.debug node_modules/@dazn/lambda-powertools-logger/index.js:82
      {"message":"published event to EventBridge","eventType":"restaurant_notified","busName":"order_events_dev","awsRegion":"us-east-1","debug-log-enabled":"true","call-chain-length":1,"level":20,"sLevel":"DEBUG"}

 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/place-order.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info functions/search-restaurants.js:33
      this is a new secret
    console.debug node_modules/@dazn/lambda-powertools-logger/index.js:82
      {"message":"finding restaurants with theme...","count":"8","theme":"cartoon","awsRegion":"us-east-1","debug-log-enabled":"false","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
    console.debug node_modules/@dazn/lambda-powertools-logger/index.js:82
      {"message":"found restaurants","count":4,"awsRegion":"us-east-1","debug-log-enabled":"false","call-chain-length":1,"level":20,"sLevel":"DEBUG"}


Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        4.667s
Ran all test suites.
```

</p></details>
