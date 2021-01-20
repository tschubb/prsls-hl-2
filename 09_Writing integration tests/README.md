# Module 9: Writing integration tests

## Add integration tests

**Goal:** Write integration tests

<details>
<summary><b>Prepare tests</b></summary><p>

1. Add a `tests` folder to the project root

2. Add a `test_cases` folder under `tests`

3. Add a `steps` folder under `tests`

4. Install `jest` as a dev dependency

`npm install --save-dev jest`

[Jest](https://jestjs.io/) is a popular test framework from Facebook, it gives you a test runner, assertion library, mocks and stubs all in one package.

5. Install `@types/jest` as a dev dependency

`npm install --save-dev @types/jest`

6. Add a `jsconfig.json` file to the project root, and paste the following into it:

```javascript
{ "typeAcquisition": { "include": [ "jest" ] } }
```

This (and combined with the `@types/jest` dev dependency) enables intellisense support for Jest in VS Code.

7. Install `cheerio` as a dev dependency

`npm install --save-dev cheerio`

[Cheerio](https://cheerio.js.org/) lets us parse the HTML returned by the `/` endpoint so we can inspect its content.

8. Install `awscred` as a dependency

`npm install --save-dev awscred`

[awscred](https://github.com/mhart/awscred) lets us resolve AWS credentials and region so that we can initialize our test environment properly - e.g. to allow `get-index` function to sign its HTTP requests with its IAM role.

9. Install `lodash` as a dependency

`npm install --save lodash`

10. Install `cross-env` as a dev dependency

`npm install --save-dev cross-env`

11. Add a file `jest.config.js` to the project root, and paste the following into the file

```javascript
module.exports = {  
  testEnvironment: 'node',
  testMatch: ['**/test_cases/**/*']
}
```

This file configures Jest, the test framework we'll be using. In this case, the `testMatch` attribute tells Jest where to find our tests.

You can read more about Jest configuration options [here](https://jestjs.io/docs/en/configuration#testtimeout-number).

</p></details>

<details>
<summary><b>Add test case for get-index</b></summary><p>

1. Add `get-index.tests.js` file under `test_cases`

2. Modify `test_cases/get-index.tests.js` to the following

```javascript
const cheerio = require('cheerio')

describe(`When we invoke the GET / endpoint`, () => {
  it(`Should return the index page with 8 restaurants`, async () => {
    const res = await when.we_invoke_get_index()

    expect(res.statusCode).toEqual(200)
    expect(res.headers['Content-Type']).toEqual('text/html; charset=UTF-8')
    expect(res.body).toBeDefined()

    const $ = cheerio.load(res.body)
    const restaurants = $('.restaurant', '#restaurantsUl')
    expect(restaurants.length).toEqual(8)
  })
})
```

Here we have a single test case that will get the response from `get-index`, inspect its status code, `Content-Type` header and the HTML content to make sure it did return 8 restaurants.

The magic, however, is in `when.we_invoke_get_index`, beause it's abstracted away and doesn't specify HOW we invoke `get-index`, it allows us to reuse this test case later as an acceptance test where we'll invoke `get-index` by calling the deployed HTTP GET `/` endpoint.

But for now, as an integration test, we'll invoke the handler code locally.

First, let's define the `when` module.

3. Add `when.js` file under `steps`

4. Modify `steps/when.js` to the following

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')

const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.Content-Type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}

const we_invoke_get_index = () => viaHandler({}, 'get-index')

module.exports = {
  we_invoke_get_index
}
```

As you can see, the `viaHandler` requires the `/functions/get-index.handler` function and calls it with the event payload `{}`, and an empty context object `{}`.

And to make it easier to validate the response, it also parses JSON response body if the `Content-Type` header is `application/json` or omitted (which would default to `application/json` anyway).

The reason why we're JSON parsing body is also to mirror the behaviour of the HTTP client `axios`, which we'll use later when implementing our acceptance tests.

5. Modify `test_cases/get-index.tests.js` to require the `when` module

```javascript
const cheerio = require('cheerio')
const when = require('../steps/when')

describe(`When we invoke the GET / endpoint`, () => {
```

6. Modify the `package.json` and add `dotEnv` and `test` script

```json
"scripts": {
  "sls": "serverless",
  "dotEnv": "sls export-env",
  "test": "npm run dotEnv && jest"
},
```

This way, whenever we run `npm run test` (which you can also use the shorthand `npm t`), we'll generate the `.env` file first, ensuring that we have the latest environment variables for our tests.

7. Run the integration test

`npm run test`

and see that the test fails with the error

```
FAIL  tests/test_cases/get-index.tests.js
  When we invoke the GET / endpoint
    ✕ Should return the index page with 8 restaurants (92ms)

  ● When we invoke the GET / endpoint › Should return the index page with 8 restaurants

    TypeError [ERR_INVALID_ARG_TYPE]: The "url" argument must be of type string. Received type undefined

      16 | const getRestaurants = async () => {
      17 |   console.log(`loading restaurants from ${restaurantsApiRoot}...`)
    > 18 |   const url = URL.parse(restaurantsApiRoot)
         |                   ^
      19 |   const opts = {
      20 |     host: url.hostname,
      21 |     path: url.pathname

      at getRestaurants (functions/get-index.js:18:19)
      at handler (functions/get-index.js:33:29)
      at viaHandler (tests/steps/when.js:8:26)
      at Object.we_invoke_get_index (tests/steps/when.js:16:35)
      at Object.it (tests/test_cases/get-index.tests.js:6:28)
```

This is because the `get-index` function needs a number of environment variables, including the URL to the `get-restaurants` endpoint. We haven't set these up in our tests.

So what we can do, is to encapsulate all the initialization logic for our tests into its own module.

Luckily, we can use the `serverless-export-env` plugin to exports all the environment variables we need to a `.env` file, and use it to initialize our tests.

8. Add `init.js` under `steps` folder.

9. Modify `init.js` to the following

```javascript
const { promisify } = require('util')
const awscred = require('awscred')
require('dotenv').config()

let initialized = false

const init = async () => {
  if (initialized) {
    return
  }
  
  const { credentials, region } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID     = credentials.accessKeyId
  process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey
  process.env.AWS_REGION            = region

  if (credentials.sessionToken) {
    process.env.AWS_SESSION_TOKEN = credentials.sessionToken
  }

  console.log('AWS credential loaded')

  initialized = true
}

module.exports = {
  init
}
```

As you can see, in addition to loading the environment variables from the `.env` file (with the line `require('dotenv').config()`), the `init` method also resolves the AWS credentials using the `awscred` module and puts the access key and secret into the environment variables.

This is so that the `get-index` function is able to use them to sign the HTTP request to the `/restaurants` endpoint.

This block of code is necessary to cater for when you're authenticated as an IAM role (instead of an IAM user).

```javascript
if (credentials.sessionToken) {
  process.env.AWS_SESSION_TOKEN = credentials.sessionToken
}
```

10. Modify `test_cases/get-index.tests.js` to require the `init` module at the top of the file

```javascript
const cheerio = require('cheerio')
const when = require('../steps/when')
const { init } = require('../steps/init')

describe(`When we invoke the GET / endpoint`, () => {
```

11. Modify `test_cases/get-index.tests.js` to add a `before` step in the test case `When we invoke the GET / endpoint`

```javascript
describe(`When we invoke the GET / endpoint`, () => {
  beforeAll(async () => await init())

  it(`Should return the index page with 8 restaurants`, async () => {
```

So that we will run the initialization logic before test case.

12. Run the integration test

`npm run test`

and see that the test still fails! This time with a different error.

```
FAIL  tests/test_cases/get-index.tests.js
  When we invoke the GET / endpoint
    ✕ Should return the index page with 8 restaurants (50ms)

  ● When we invoke the GET / endpoint › Should return the index page with 8 restaurants

    TypeError: Cannot read property 'replace' of null

      at dispatchHttpRequest (node_modules/axios/lib/adapters/http.js:84:74)
      at httpAdapter (node_modules/axios/lib/adapters/http.js:21:10)
      at dispatchRequest (node_modules/axios/lib/core/dispatchRequest.js:52:10)

  console.log tests/steps/init.js:25
    AWS credential loaded

  console.log functions/get-index.js:17
    loading restaurants from https://#{ApiGatewayRestApi}.execute-api.#{AWS::Region}.amazonaws.com/dev/restaurants...

Test Suites: 1 failed, 1 total
Tests:       1 failed, 1 total
Snapshots:   0 total
Time:        0.947s, estimated 1s
```

Notice that weird URL that's logged from the `get-index` function?

```
  console.log functions/get-index.js:17
    loading restaurants from https://#{ApiGatewayRestApi}.execute-api.#{AWS::Region}.amazonaws.com/dev/restaurants...
```

That's literal value we gave to the `get-index` function's `restaurants_api` environment variable.

```yml
get-index:
  handler: functions/get-index.handler
  events:
    - http:
        path: /
        method: get
  environment:
    restaurants_api: https://#{ApiGatewayRestApi}.execute-api.#{AWS::Region}.amazonaws.com/${self:provider.stage}/restaurants
    cognito_user_pool_id: !Ref CognitoUserPool
    cognito_client_id: !Ref WebCognitoUserPoolClient
```

Welcome to the real world, where your tools don't integrate perfectly with each other...

![](/images/mod09-001.gif)

</p></details>

<details>
<summary><b>Compromises, compromises...</b></summary><p>

Ok, so the `serverless-export-env` plugin does 90% of the work and gives us the environment variables we need, but it falls down when it comes to the `restaurants_api` environment variable because it doesn't work with the `serverless-pseudo-parameters`.

To solve that, we have to drop back down to using CloudFormation pseudo functions.

Why can't we have all the nice things?

1. In the `serverless.yml`, find the `functions.get-index` block, and change the `restaurants_api` environment variable to the following:

```yml
restaurants_api:
  Fn::Join:
    - ""
    - - https://
      - !Ref ApiGatewayRestApi
      - .execute-api.${self:provider.region}.amazonaws.com/${self:provider.stage}/restaurants
```

After your change, the `get-index` function should look like this:

```yml
get-index:
  handler: functions/get-index.handler
  events:
    - http:
        path: /
        method: get
  environment:
    restaurants_api:
      Fn::Join:
        - ""
        - - https://
          - !Ref ApiGatewayRestApi
          - .execute-api.${self:provider.region}.amazonaws.com/${self:provider.stage}/restaurants
    cognito_user_pool_id: !Ref CognitoUserPool
    cognito_client_id: !Ref WebCognitoUserPoolClient
```

2. Rerun the test

`npm run test`

and you should see the test is now finally passing!

```
PASS  tests/test_cases/get-index.tests.js
  When we invoke the GET / endpoint
    ✓ Should return the index page with 8 restaurants (517ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        1.162s, estimated 2s
```

Congratulation! You have just written and passed your first integration test!

3. But notice all those `console.log` messages are clutering in the output. If you want to 'silence' them then add this line to the top of `get-index.tests.js`:

`console.log = jest.fn()`

`jest.fn()` creates a mock function (which you can read all about [here](https://jestjs.io/docs/en/mock-functions.html)). So this in case, we're monkey-patching the `console.log` method with a mock function so we don't have to see those console logs.

</p></details>

<details>
<summary><b>Add test case for get-restaurants</b></summary><p>

Now let's do more of the same and add a test case for the `get-restaurants` function.

1. Add `get-restaurants.tests.js` under `test_cases`

2. Modify `test_cases/get-restaurants.tests.js` to the following

```javascript
const { init } = require('../steps/init')
const when = require('../steps/when')
console.log = jest.fn()

describe(`When we invoke the GET /restaurants endpoint`, () => {
  beforeAll(async () => await init())

  it(`Should return an array of 8 restaurants`, async () => {
    const res = await when.we_invoke_get_restaurants()

    expect(res.statusCode).toEqual(200)
    expect(res.body).toHaveLength(8)

    for (let restaurant of res.body) {
      expect(restaurant).toHaveProperty('name')
      expect(restaurant).toHaveProperty('image')
    }
  })
})
```

Once again, we're using `when.we_invoke_get_restaurants` to abstract away HOW we invoke the `get-restaurants` function.

In this test, we check that 8 restaurants were returned, and that each has the properties `name` and `image`.

But first, let's make sure `when.we_invoke_get_restaurants` exists.

3. Modify `when.js` to add a `we_invoke_get_restaurants` function after `we_invoke_get_index`

```javascript
const we_invoke_get_restaurants = () => viaHandler({}, 'get-restaurants')
```

and then add it to `module.exports`, i.e.

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants
}
```

4. Run the integration test

`npm run test`

and see that both tests pass

```
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js

Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.458s, estimated 2s
```

</p></details>

<details>
<summary><b>Add test case for search-restaurants</b></summary><p>

1. Add `search-restaurants.tests.js` under `test_cases`

2. Modify `test_cases/search-restaurants.tests.js` to the following

```javascript
const { init } = require('../steps/init')
const when = require('../steps/when')
console.log = jest.fn()

describe(`When we invoke the POST /restaurants/search endpoint with theme 'cartoon'`, () => {
  beforeAll(async () => await init())

  it(`Should return an array of 4 restaurants`, async () => {
    let res = await when.we_invoke_search_restaurants('cartoon')

    expect(res.statusCode).toEqual(200)
    expect(res.body).toHaveLength(4)

    for (let restaurant of res.body) {
      expect(restaurant).toHaveProperty('name')
      expect(restaurant).toHaveProperty('image')
    }
  })
})
```

3. Modify `when.js` to add a `we_invoke_search_restaurants` function after `we_invoke_get_restaurants`

```javascript
const we_invoke_search_restaurants = theme => {
  let event = { 
    body: JSON.stringify({ theme })
  }
  return viaHandler(event, 'search-restaurants')
}
```

and add the new `we_invoke_search_restaurants` function to `module.exports`

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants
}
```

4. Run the integration test

`npm run test`

and see that all three tests pass

```
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js

Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        1.609s, estimated 2s
```

Well done!

</p></details>

## Challenges

Before we move on to acceptance tests, I would like you to take a moment and think about what we're doing and how we can improve things.

For starters, did you notice that for the `get-index` and `get-restaurants` tests, we depend on the current state of the database?

If we hadn't seeded the database with 8 restaurants, the tests would fail.

How can we improve the tests (and maybe our implementation too?) so that they're self-contained and can be executed without making assumptions about what data we current have in the database?

And, so far we only have 1 test case for each function. It's sufficient to illustrate how to write integration tests, but it's anything but comprehensive. What other test cases would you add?
