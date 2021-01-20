# Module 21: Structured logging

## Apply structured logging to the project

<details>
<summary><b>Using a simple logger</b></summary><p>

I built a suite of tools to help folks build production-ready serverless applications while I was at DAZN. It's now open source: [dazn-lambda-powertools](https://github.com/getndazn/dazn-lambda-powertools).

One of the tools available is a very simple logger that supports structured logging (amongst other things).

So, first, let's install the logger for our project.

1. At the project root, run the command `npm install --save @dazn/lambda-powertools-logger` to install the logger.

Now we need to change all the places where we're using `console.log`.

2. Open `functions/get-index.js` and add the following to the top of the file

```javascript
const Log = require('@dazn/lambda-powertools-logger')
```

on ln20, replace

```javascript
console.log(`loading restaurants from ${restaurantsApiRoot}...`)
```

with

```javascript
Log.debug('getting restaurants...', { url: restaurantsApiRoot })
```

Notice that the `restaurantsApiRoot` is captured as a separate `url` attribute in the log message. Capturing variables as attributes (instead of baking them into the message) makes them easier to search and filter by.

On ln37, replace

```javascript
console.log(`found ${restaurants.length} restaurants`)
```

with

```javascript
Log.debug('got restaurants', { count: restaurants.length })
```

Again, notice how `count` is captured as a separate attribute.

3. Open `functions/get-restaurants.js` and add the following to the top of the file

```javascript
const Log = require('@dazn/lambda-powertools-logger')
```

On ln11, replace

```javascript
console.log(`fetching ${count} restaurants from ${tableName}...`)
```

with

```javascript
Log.debug('getting restaurants from DynamoDB...', {
  count,
  tableName
})
```

And then on ln22, replace

```javascript
console.log(`found ${resp.Items.length} restaurants`)
```

with

```javascript
Log.debug('found restaurants', {
  count: resp.Items.length
})
```

4. Open `functions/place-order.js` and add the following to the top of the file

```javascript
const Log = require('@dazn/lambda-powertools-logger')
```

On ln12, replace

```javascript
console.log(`placing order ID [${orderId}] to [${restaurantName}]`)
```

with

```javascript
Log.debug('placing order...', { orderId, restaurantName })
```

Similarly, on ln26, replace

```javascript
console.log(`published 'order_placed' event into EventBridge`)
```

with

```javascript
Log.debug(`published event into EventBridge`, {
  eventType: 'order_placed',
  busName
})
```

5. Repeat the same process for `functions/notify-restaurant` and `functions/search-restaurants`, using your best judgement on what information you should log in each case.

6. Run the integration tests

`npm run test`

and see that the functions are now logging in JSON

```
 
 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.debug
      {"message":"getting restaurants...","url":"https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/restaurants","awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

    console.debug
      {"message":"got restaurants","count":8,"awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.debug
      {"message":"getting restaurants from DynamoDB...","count":"8","tableName":"workshop-yancui-dev-RestaurantsTable-N9HWPJCPE2EW","awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

    console.debug
      {"message":"found restaurants","count":8,"awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

 PASS  tests/test_cases/notify-restaurant.tests.js
  ● Console

    console.debug
      {"message":"notified restaurant of order","orderId":"62cdbd2d-325f-5f21-9341-cb22e7da6c27","restaurantName":"Fangtasia","awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

    console.debug
      {"message":"published event into EventBridge","eventType":"restaurant_notified","busName":"order_events_dev","awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)


ReferenceError: You are trying to `import` a file after the Jest environment has been torn down.

      at Object.userAgent (node_modules/aws-sdk/lib/util.js:34:43)
      at HttpRequest.setUserAgent (node_modules/aws-sdk/lib/http.js:111:78)
      at new HttpRequest (node_modules/aws-sdk/lib/http.js:104:10)
      at new Request (node_modules/aws-sdk/lib/request.js:328:24)
      at features.constructor.makeRequest (node_modules/aws-sdk/lib/service.js:202:19)
 PASS  tests/test_cases/place-order.tests.js
  ● Console

    console.debug
      {"message":"placing order...","orderId":"38fe7f3a-b6bd-5ecd-94c2-afdb119746c1","restaurantName":"Fangtasia","awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

    console.debug
      {"message":"published event into EventBridge","eventType":"order_placed","busName":"order_events_dev","awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)


ReferenceError: You are trying to `import` a file after the Jest environment has been torn down.

      at Object.userAgent (node_modules/aws-sdk/lib/util.js:34:43)
      at HttpRequest.setUserAgent (node_modules/aws-sdk/lib/http.js:111:78)
      at new HttpRequest (node_modules/aws-sdk/lib/http.js:104:10)
      at new Request (node_modules/aws-sdk/lib/request.js:328:24)
      at features.constructor.makeRequest (node_modules/aws-sdk/lib/service.js:202:19)
 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info
      this is a secret

      at Function.module.exports.handler.middy (functions/search-restaurants.js:32:11)

    console.debug
      {"message":"finding restaurants with theme","count":"8","theme":"cartoon","tableName":"workshop-yancui-dev-RestaurantsTable-N9HWPJCPE2EW","awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

    console.debug
      {"message":"found restaurants","count":4,"awsRegion":"us-east-1","level":20,"sLevel":"DEBUG"}

      at Logger.log (node_modules/@dazn/lambda-powertools-logger/index.js:82:30)

A worker process has failed to exit gracefully and has been force exited. This is likely caused by tests leaking due to improper teardown. Try running with --runInBand --detectOpenHandles to find leaks.

Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        5.271 s, estimated 14 s
Ran all test suites.
```

</p></details>

<details>
<summary><b>Disable debug logging in production</b></summary><p>

This logger allows you to control the default log level via the `LOG_LEVEL` environment variable. Let's configure the `LOG_LEVEL` environment such that we'll be logging at `INFO` level in production, but logging at `DEBUG` level everywhere else.

1. Open `serverless.yml`. Under the `custom` section at the top, add `stage` and `logLevel` as below:

```yml
stage: ${opt:stage, self:provider.stage}
logLevel:
  prod: INFO
  default: DEBUG
```

`custom.stage` uses the `${xxx, yyy}` syntax to provide a fall back. In this case, we're saying "if a `stage` variable is provided via the CLI, e.g. `sls deploy --stage staging`, then resolve to `staging`; otherwise, fallback to `provider.stage` in this file (hence the `self` reference"

2. Still in the `serverless.yml`, under `provider.environment` section, add the following

```yml
LOG_LEVEL: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

This uses the same `${xxx, yyy}` syntax as before.

After this change, the `provider` section should look like this:

```yml
provider:
  name: aws
  runtime: nodejs12.x
  environment:
    serviceName: ${self:service}
    stage: ${self:provider.stage}
    LOG_LEVEL: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

This applies the `LOG_LEVEL` environment variable (used to decide what level the logger should log at) to all the functions in the project (since it's specified under `provider`).

It references the `custom.logLevel` object (with the `self:` syntax), and also references the `custom.stage` value (remember, this can be overriden by CLI options). So when the deployment stage is `prod`, it resolves to `self:custom.logLevel.prod` and `LOG_LEVEL` would be set to `INFO`.

The second argument, `self:custom.logLevel.default` provides the fallback if the first path is not found. If the deployment stage is `dev`, it'll see that `self:custom.logLevel.dev` doesn't exist, and therefore use the fallback `self:custom.logLevel.default` and set `LOG_LEVEL` to `DEBUG` in that case.

This is a nice trick to specify a stage-specific override, but then fall back to some default value otherwise.

3. Throughput the `serverless.yml` we have references to `provider.stage` directly. If we want to deploy to another stage, we'll need to change these references to use `${self:custom.stage}` instead.

Find and replace `${self:provider.stage}` in the `serverless.yml` with `${self:custom.stage}`.

</p></details>
