# Module 22: Sample debug logs in production

## Sample 1% of debug logs in production

<details>
<summary><b>Install lambda-powertools-pattern-basic</b></summary><p>

1. At the project root, run the command `npm install --save @dazn/lambda-powertools-pattern-basic`.

This package gives you a simple wrapper which applies a couple of [middy](https://github.com/middyjs/middy) middlewares for your function:

* `@dazn/lambda-powertools-middleware-sample-logging`: which supports sampling debug logs. The wrapper configures this sample logging middleware to sample debug logs for 1% of invocations.

* `@dazn/lambda-powertools-middleware-correlation-ids`: which extracts correlation IDs from the invocation event and makes them available for the logger. It also supports a special correlation ID `debug-log-enabled`, which enables sampling debug logs at the user transaction (a chain of Lambda invocations) level.

* `@dazn/lambda-powertools-middleware-log-timeout`: which emits an error message for when a function times out. Normally, when a Lambda function times out, you don't get an error message from the application, which makes debugging time out errors difficult.

Now we need to apply it to all of our functions.

</p></details>

<details>
<summary><b>Wrap function handlers</b></summary><p>

1. Open `functions/get-index.js` and require the `@dazn/lambda-powertools-pattern-basic` module (at the top of the file)

```javascript
const wrap = require('@dazn/lambda-powertools-pattern-basic')
```

And use it to wrap our handler function.

On ln35, change

```js
module.exports.handler = async (event, context) => {
```

to:

```javascript
module.exports.handler = wrap(async (event, context) => {
```

and don't forget to add the closing `)` on ln58!

This works exactly like the Middy and SSM middleware we saw earlier, because under the hood, the `@dazn/lambda-powertools-pattern-basic` wrapper also uses `middy`.

2. Repeat step 1 for `notify-restaurant` and `place-order` functions.

3. Open `functions/get-restaurants.js` and require the `@dazn/lambda-powertools-pattern-basic` module (at the top of the file)

```javascript
const wrap = require('@dazn/lambda-powertools-pattern-basic')
```

And use it to wrap our handler function. On ln28 replace

```js
module.exports.handler = middy(async (event, context) => {
```

with

```js
module.exports.handler = wrap(async (event, context) => {
```

The SSM would still continue to work because the `@dazn/lambda-powertools-pattern-basic` module's wrapper also uses Middy.

Also, we can remove the direct dependency on Middy in this module.

Remove this line from the file:

```js
const middy = require('@middy/core')
```

4. Repeat step 3 for `search-restaurants` function as well.

5. Run integration test

`npm run test`

and see that all the tests are still passing.

4. Deploy the project

`npx sls deploy`

</p></details>

<details>
<summary><b>Configure the default sample rate</b></summary><p>

One of the things the `@dazn/lambda-powertools-pattern-basic` wrapper gives you is basic debug log sampling - that is, even when you log at a higher level, say `INFO`, where you shouldn't see `DEBUG` logs, it'll sample your `DEBUG` logs for 1% of the invocations.

This ensures that even in production environment, where it's too costly to log all debug messages, you can still retain a small sample of debug messages for when a problem arise and you need to investigate it.

To see this sampling behaviour in action, you can either deploy to a `prod` stage, or you can change the default log level for our `dev` stage. For simplicity (and to avoid the hassle of setting up those SSM parameters for another stage), let's do that.

1. If you change the default log level to `INFO`, i.e. in the `serverless.yml`, change `custom.logLevel` to:

```yml
logLevel:
  prod: ERROR
  default: INFO
```

then redeploy

`npx sls deploy`

2. Now reload the homepage a few times and you'll notice that you no longer see the debug log messages in the `get-index` and `get-restaurants` functions' logs.

This is because by default the `@dazn/lambda-powertools-pattern-basic` wrapper configures the debug logs sample rate to be 1% based on industry average.

Now, try turning this up to say, 10%, by adding a `SAMPLE_DEBUG_LOG_RATE` environment variable.

In `serverless.yml`, and add a `SAMPLE_DEBUG_LOG_RATE` environment variable under `provider.environment`:

```yml
environment:
  SAMPLE_DEBUG_LOG_RATE: 0.1
  ...
```

then redeploy

`npx sls deploy`

3. Now reload the homepage a few more times, and you should occassionally see debug log messages in the logs for the `get-index` and `get-restaurants` functions.

![](/images/mod22-001.png)

</p></details>
