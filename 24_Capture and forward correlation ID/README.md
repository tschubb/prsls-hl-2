# Module 24: Capture and forward correlation ID

## Capture and forward correlation IDs

Since we're using the `@dazn/lambda-powertools-pattern-basic` and `@dazn/lambda-powertools-logger`, we are already capturing incoming correlation IDs and including them in the logs.

However, we need to make sure the `get-index` function forwards the correlation IDs along, while still using the X-Ray instrumented `axios` module.

The `dazn-lambda-powertools` project also has wrappers for `EventBridge`, `SNS` and many other AWS SDK clients that will automatically forward captured correlation IDs. We can use them as stand-in replacements for the `EventBridge` and `SNS` clients from the AWS SDK.

<details>
<summary><b>Forward correlation IDs from get-index</b></summary><p>

You can access the auto-captured correlation IDs using the `@dazn/lambda-powertools-correlation-ids` package. From here, we can include them as HTTP headers.

1. At the project root, run `npm install --save @dazn/lambda-powertools-correlation-ids`.

2. Open `functions/get-index.js` and require the `@dazn/lambda-powertools-correlation-ids` module (at the top of the file).

```javascript
const CorrelationIds = require('@dazn/lambda-powertools-correlation-ids')
```

3. Staying in `functions/get-index.js`, on ln 32, replace

```javascript
const httpReq = http.get(restaurantsApiRoot, {
  headers: opts.headers
})
```

with the following:

```javascript
const httpReq = http.get(restaurantsApiRoot, {
  headers: Object.assign({}, opts.headers, CorrelationIds.get())
})
```

This adds the correlation IDs (that has been automatically extracted and captured from the invocation event) to the HTTP headers.

These changes are enough to ensure correlation IDs are included in the HTTP headers in the request to the `GET /restaurants` endpoint.

On the other side of the HTTP request, the `get-restaurants` function would extract them and include them in its logs. You don't need to do anything yourself, as the `@dazn/lambda-powertools-pattern-basic` wrapper would extract the correlation IDs out for you.

4. Open `tests/steps/when.js` and modify the `viaHandler` method so that `context` is initialized with a `awsRequestId`, e.g.

`const context = { awsRequestId: 'test' }`

This will be used to initialize the correlation ID with and you'll see it in the logs when you run the test (pay attention to the `x-correlation-id` field in the log messages)

`npm run test`

```
 PASS  tests/test_cases/notify-restaurant.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/place-order.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info functions/search-restaurants.js:33
      this is a new secret
    console.debug node_modules/@dazn/lambda-powertools-logger/index.js:82
      {"message":"finding restaurants with theme...","count":"8","theme":"cartoon","awsRegion":"us-east-1","awsRequestId":"test","x-correlation-id":"test","debug-log-enabled":"true","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
    console.debug node_modules/@dazn/lambda-powertools-logger/index.js:82
      {"message":"found restaurants","count":4,"awsRegion":"us-east-1","awsRequestId":"test","x-correlation-id":"test","debug-log-enabled":"true","call-chain-length":1,"level":20,"sLevel":"DEBUG"}


Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        4.283s
```

5. Redeploy the project.

`npx sls deploy`

6. Once the deployment is done, load the page. And then open the X-Ray console to make sure that the X-Ray tracing is still working.

7. Open the CloudWatch console to check the logs for both `get-index` and `get-restaurants`. You should see that the same correlation ID is included in both logs.

![](/images/mod24-001.png)

![](/images/mod24-002.png)

</p></details>

<details>
<summary><b>Forward correlation IDs through EventBridge events</b></summary><p>

The `dazn-lambda-powertools` suite has a number of like-for-like replacement packages for AWS SDK clients that can automatically forward correlation IDs that has been captured by the wrapper.

So, to forward correlation IDs from the `place-order` function onto `notify-restaurant` through EventBridge, we need to use the `@dazn/lambda-powertools-eventbridge-client` NPM package.

1. Install the `@dazn/lambda-powertools-eventbridge-client`

`npm install @dazn/lambda-powertools-eventbridge-client`

2. Open `functions/place-order.js` file, replace

```javascript
const eventBridge = XRay.captureAWSClient(new EventBridge())
```

with this:

```javascript
const eventBridge = XRay.captureAWSClient(require('@dazn/lambda-powertools-eventbridge-client'))
```

You can also remove this line too, since we no longer need it:

```javascript
const EventBridge = require('aws-sdk/clients/eventbridge')
```

3. Repeat step 2 for `functions/notify-restaurant.js`

4. Redeploy

`npx sls deploy`

and place a few orders, and then check the logs for `place-order` and `notify-restaurant`. You should see on a few occassions (remember, debug logs are sampled at 10%) that debug logs are enabled on the whole transaction and that both functions have the same `x-correlation-id`.

e.g. in the `place-order` function:
![](/images/mod24-003.png)

the same `x-correlation-id` is recorded in `notify-restaurant` function, and note the decision to enable debug logging is passed along
![](/images/mod24-004.png)

4. Also, check out [**this repo**](https://github.com/theburningmonk/lambda-distributed-tracing-demo/tree/master/lambda-powertools) to see a more comprehensive demo of how you can auto-extract and forward correlation IDs through a variety of different event sources with the dazn-lambda-powertools.

</p></details>

<details>
<summary><b>Adding custom correlation IDs</b></summary><p>

By default, the Lambda powertools would initialize a correlation ID for you using either:

* API Gateway request ID for API Gateway events
* Lambda request ID for all other events

The reason it prefers the API Gateway request ID where applicable is that, unlike the Lambda request ID, the API Gateway request ID is returned to the caller as a HTTP response header. This means, in the event of an error, the caller can report this ID to the end-user, and inform him/her to contact your customer support team with the ID (or just the first 6 characters of it).

This lets you find the relevant logs quicker as you can search for logs whose `x-correlation-id` matches the ID the customer has provided.

HOWEVER, this is not always possible. And we can do ourselves a massive favour by tracking a few other correlation IDs - that is, IDs that we want to correlate logs from different functions with.

For example, in the order flow, some natural candidates are:

* user ID
* order ID
* restaurant name

So, let's capture these as correlation IDs in the `place-order` function, which then propagates them across other functions down the line.

1. Open `functions/place-order.js`, and require the `@dazn/lambda-powertools-correlation-ids` package at the top

```javascript
const CorrelationIds = require('@dazn/lambda-powertools-correlation-ids')
```

2. At ln14, before we log the debug message

```javascript
Log.debug('placing order...', { orderId, restaurantName })
```

Let's add the `orderId`, `userId` and `restaurantName` as correlation IDs, which would be automatically included in our logs, so we wouldn't need to explicitly include them each time.

Replace ln14, `Log.debug('placing order...', { orderId, restaurantName })` with the following

```javascript
const userId = event.requestContext.authorizer.claims.sub
CorrelationIds.set('userId', userId)
CorrelationIds.set('orderId', orderId)
CorrelationIds.set('restaurantName', restaurantName)
Log.debug('placing order...')
```

Yup, you can access the authenticated user's information through `event.requestContext.authorizer.claims`. In there you can find the user's username too, which based on our Cognito User Pool configuration, is the user's email. And given GDPR and other similar rules, you shouldn't include PII data like email in the logs! `sub` is the `subject` in JWT (see [here](https://en.wikipedia.org/wiki/JSON_Web_Token)) and in this case, a unique identifier for a user in Cognito, whereas usernames can be reassigned, `sub` is never reassigned.

3. Open `functions/notify-restaurant.js`, on ln19, replace

```javascript
const { restaurantName, orderId } = order
Log.debug('notified restaurant', {
  orderId,
  restaurantName
})
```

with

```javascript
Log.debug('notified restaurant')
```

We don't need to explicitly set them anymore, they'll come through as correlation IDs regardless.

![](/images/mod24-005.png)

![](/images/mod24-006.png)

</p></details>
