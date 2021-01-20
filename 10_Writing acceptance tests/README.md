# Module 10: Writing acceptance tests

## Add acceptance tests

**Goal:** Write acceptance tests

<details>
<summary><b>Prepare for acceptance tests</b></summary><p>

1. Open `steps/when.js`.

2. First, we need to add a couple of dependencies, let's add them to the top of the file.

```javascript
const aws4 = require('aws4')
const URL = require('url')
const http = require('axios')
```

These are necessary because we'll need to make HTTP request to the deploy API endpoints, and where applicable we might need to sign the request with our IAM credentials (e.g. with the `/restaurants` endpoint) which is why we need `aws4`.

To allow the `when` module to toggle between "invoke function locally" and "call the deployed API", we can use an environment variable that is set when we run the test.

Let's call this environment variable `TEST_MODE`.

So let's add this line to the top of the `when.js` file:

```javascript
const mode = process.env.TEST_MODE
```

And let's say if `mode` is `handler`, then we'll invoke the functions locally, otherwise, if it's `http`, then we'll call the deploy API endpoint.

2. With that in mind, let's modify the `when.we_invoke_get_index` function to toggle between invoking function locally and remotely.

```javascript
const we_invoke_get_index = async () => {
  switch (mode) {
    case 'handler':
      return await viaHandler({}, 'get-index')
    case 'http':
      return await viaHttp('', 'GET')
    default:
      throw new Error(`unsupported mode: ${mode}`)
  }
}
```

Don't worry, we'll implement the `viaHttp` method in a minute, for now, just know that we need to pass in a relative path (in this case `''` for the root) and a HTTP method (`'GET'` in this case).

To make the HTTP request, we need to know the root URL for the deploy API and put it somewhere so the `viaHttp` method can use.

Luckily, we already capture and load environment variables via the `serverless-export-env` plugin and the `dotEnv` module.

So we just need to add the root URL to our API as an environment variable.

3. Open `serverless.yml`.

4. Add the following to the `resources.Outputs` section:

```yml
RestApiUrl:
  Value: 
    Fn::Join:
      - ""
      - - https://
        - !Ref ApiGatewayRestApi
        - .execute-api.${self:provider.region}.amazonaws.com/${self:provider.stage}
```

After this change, the `resources.Outputs` section of your `serverless.yml` should look like this (pay attention to the indentation):

```yml
  Outputs:
    RestApiUrl: ...

    RestaurantsTableName: ...

    CognitoUserPoolId: ...

    CognitoUserPoolArn: ...

    CognitoUserPoolWebClientId: ...

    CognitoUserPoolServerClientId: ...
```

This adds a `RestApiUrl` output to the CloudFormation stack, which the `serverless-export-env` plugin would capture into the `.env` file as an environment variable called `REST_API_URL`. This environment variable would then be picked up and loaded into our tests by the `init` module.

But we also need to set the `TEST_MODE` environment variable too. We'll do that in the `scripts` in `package.json`

5. Open `package.json`.

6. Change the `test` script to the following:

```json
"test": "npm run dotEnv && cross-env TEST_MODE=handler jest"
```

After this change, your `scripts` object should look like this:

```json
  "scripts": {
    "sls": "serverless",
    "dotEnv": "sls export-env",
    "test": "npm run dotEnv && cross-env TEST_MODE=handler jest"
  },
```

This sets the `TEST_MODE` environment variable to `handler` whenever we run `npm run test` (or `npm t`).

7. Rerun the integration tests

`npm run test`

and see that the tests are still passing.

```
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js
 PASS  tests/test_cases/get-index.tests.js

Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        1.74s, estimated 2s
```

Ok, now we're ready to implement the acceptance test.

</p></details>

<details>
<summary><b>Implement acceptance test for get-index</b></summary><p>

Now, let's fill in the gaps and implement the `viaHttp` method we left off in the last exercise.

1. Open `steps/when.js`.

2. Add the following after the `viaHandler` function:

```javascript
const respondFrom = async (httpRes) => ({
  statusCode: httpRes.status,
  body: httpRes.data,
  headers: httpRes.headers
})

const signHttpRequest = (url) => {
  const urlData = URL.parse(url)
  const opts = {
    host: urlData.hostname,
    path: urlData.pathname
  }

  aws4.sign(opts)
  return opts.headers
}

const viaHttp = async (relPath, method, opts) => {
  const url = `${process.env.REST_API_URL}/${relPath}`
  console.info(`invoking via HTTP ${method} ${url}`)

  try {
    const data = _.get(opts, "body")
    let headers = {}
    if (_.get(opts, "iam_auth", false) === true) {
      headers = signHttpRequest(url)
    }

    const authHeader = _.get(opts, "auth")
    if (authHeader) {
      headers.Authorization = authHeader
    }

    const httpReq = http.request({
      method, url, headers, data
    })

    const res = await httpReq
    return respondFrom(res)
  } catch (err) {
    if (err.status) {
      return {
        statusCode: err.status,
        headers: err.response.headers
      }
    } else {
      throw err
    }
  }
}
```

Let's break it down.

This `viaHttp` method makes a HTTP request to the relative path on the `REST_API_URL` environment variable (which we configured in the `serverless.yml` and loaded through `.env` file that's generated before every test).

You can pass in an `opts` object to pass in additional arguments:

* `body`: useful for `POST` and `PUT` requests.
* `iam_auth`: we should sign the HTTP request using our IAM credentials (which is what the `signHttpRequest` method is for)
* `auth`: include this as the `Authorization` header, used for authenticating against Cognito-protected endpoints (i.e. `search-restaurants`)

Since `axios` has a different response structure to our Lambda function, we need the `respondFrom` method massage the `axios` response to what we need.

Now, to actually run the acceptance test, we need to a new script to `package.json`.

3. Open `package.json`.

4. Add an `acceptance` script to `scripts`.

```json
"acceptance": "npm run dotEnv && cross-env TEST_MODE=http jest"
```

After this change, your `scripts` section should look like this:

```json
  "scripts": {
    "sls": "serverless",
    "dotEnv": "sls export-env",
    "test": "npm run dotEnv && cross-env TEST_MODE=handler jest",
    "acceptance": "npm run dotEnv && cross-env TEST_MODE=http jest"
  },
```

5. Run the acceptance test

`npm run acceptance`

and see that the `get-index` function is failing

```
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js
 FAIL  tests/test_cases/get-index.tests.js
  ● Console

    console.info tests/steps/when.js:39
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/

  ● When we invoke the GET / endpoint › Should return the index page with 8 restaurants

    expect(received).toEqual(expected) // deep equality

    Expected: "text/html; charset=UTF-8"
    Received: undefined

      11 |
      12 |     expect(res.statusCode).toEqual(200)
    > 13 |     expect(res.headers['Content-Type']).toEqual('text/html; charset=UTF-8')
         |                                         ^
      14 |     expect(res.body).toBeDefined()
      15 |
      16 |     const $ = cheerio.load(res.body)

      at Object.it (tests/test_cases/get-index.tests.js:13:41)

Test Suites: 1 failed, 2 passed, 3 total
Tests:       1 failed, 2 passed, 3 total
Snapshots:   0 total
Time:        1.933s, estimated 2s
```

This is because the HTTP client `axios` lower-cases the `Content-Type` automatically, but both our test and the `get-index` function is returning camel-case.

At this point, you have two options:

1. in the `respondFrom` function, change the `Content-Type` header to be camel-case
2. change the `viaHandler` method and the `get-index` function to use lower case `content-type` to match `axios`'s behaviour

While I don't have a strong preference either way, I opted for option 2.

6. Modify `test_cases/get-index.tests.js` to look for `content-type` instead of `Content-Type`

```javascript
expect(res.headers['content-type']).toEqual('text/html; charset=UTF-8')
```

7. Modify `functions/get-index.js` to return `content-type` header instead of `Content-Type`

```javascript
const response = {
  statusCode: 200,
  headers: {
    'content-type': 'text/html; charset=UTF-8'
  },
  body: html
}
```

8. Modify `steps/when.js`'s `viaHandler` method to look for `content-type` instead of `Content-Type`

```javascript
const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

9. Run the acceptance test

`npm run acceptance`

and see that the `get-index` function is now passing

```
 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info tests/steps/when.js:39
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/

 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js

Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        1.924s, estimated 2s
```

</p></details>

<details>
<summary><b>Implement acceptance test for get-restaurants</b></summary><p>

1. Modify `when.we_invoke_get_restaurants` to toggle between invoking function locally and remotely

```javascript
const we_invoke_get_restaurants = async () => {
  switch (mode) {
    case 'handler':
      return await viaHandler({}, 'get-restaurants')
    case 'http':
      return await viaHttp('restaurants', 'GET', { iam_auth: true })
    default:
      throw new Error(`unsupported mode: ${mode}`)
  }
}
```

2. Run the acceptance test

`npm run acceptance`

and see that both `get-index` and `get-restaurants` tests are passing

```
 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:39
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaur
ants

 PASS  tests/test_cases/search-restaurants.tests.js
 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info tests/steps/when.js:39
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/


Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        1.707s, estimated 2s
```

</p></details>

<details>
<summary><b>Implement acceptance test for search-restaurants</b></summary><p>

This is a bit trickier, since the `search-restaurant` endpoint is protected by a Cognito custom authorizer. It means our test code would need to authenticate itself against Cognito first.

Remember that `server` client we created when set up the Cognito user pool? If not, look in your `serverless.yml` you'll find it.

```yml
ServerCognitoUserPoolClient:
  Type: AWS::Cognito::UserPoolClient
  Properties:
    ClientName: server
    UserPoolId: !Ref CognitoUserPool
    ExplicitAuthFlows:
      - ALLOW_ADMIN_USER_PASSWORD_AUTH
      - ALLOW_REFRESH_TOKEN_AUTH
    PreventUserExistenceErrors: ENABLED
```

The `ALLOW_ADMIN_USER_PASSWORD_AUTH` auth flow allows us to call the Cognito admin endpoints to register users and sign in as them.

Oh, and to avoid having an implicit dependency on some user having been created in Cognito, each test should create its own user, and delete it afterwards.

And to avoid clashing on username, let's use randomized usernames.

1. Install `chance` as a dependency

`npm install --save chance`

[Chance](https://chancejs.com/) is a handy module for generating random strings, names, etc. We'll use it to generate usernames, first names and last names, etc.

Notice that I'm installing it as a production dependency, and not a dev dependency? It's because we're going to use it in later exercises too ;-)

2. Add a file `given.js` to `steps` folder

3. Modify `steps/given.js` to the following

```javascript
const AWS = require('aws-sdk')
const chance  = require('chance').Chance()

// needs number, special char, upper and lower case
const random_password = () => `${chance.string({ length: 8})}B!gM0uth`

const an_authenticated_user = async () => {
  const cognito = new AWS.CognitoIdentityServiceProvider()
  
  const userpoolId = process.env.COGNITO_USER_POOL_ID
  const clientId = process.env.COGNITO_USER_POOL_SERVER_CLIENT_ID

  const firstName = chance.first({ nationality: "en" })
  const lastName  = chance.last({ nationality: "en" })
  const suffix    = chance.string({length: 8, pool: "abcdefghijklmnopqrstuvwxyz"})
  const username  = `test-${firstName}-${lastName}-${suffix}`
  const password  = random_password()
  const email     = `${firstName}-${lastName}@big-mouth.com`

  const createReq = {
    UserPoolId        : userpoolId,
    Username          : username,
    MessageAction     : 'SUPPRESS',
    TemporaryPassword : password,
    UserAttributes    : [
      { Name: "given_name",  Value: firstName },
      { Name: "family_name", Value: lastName },
      { Name: "email",       Value: email }
    ]
  }
  await cognito.adminCreateUser(createReq).promise()

  console.log(`[${username}] - user is created`)
  
  const req = {
    AuthFlow        : 'ADMIN_NO_SRP_AUTH',
    UserPoolId      : userpoolId,
    ClientId        : clientId,
    AuthParameters  : {
      USERNAME: username,
      PASSWORD: password
    }
  }
  const resp = await cognito.adminInitiateAuth(req).promise()

  console.log(`[${username}] - initialised auth flow`)

  const challengeReq = {
    UserPoolId          : userpoolId,
    ClientId            : clientId,
    ChallengeName       : resp.ChallengeName,
    Session             : resp.Session,
    ChallengeResponses  : {
      USERNAME: username,
      NEW_PASSWORD: random_password()
    }
  }
  const challengeResp = await cognito.adminRespondToAuthChallenge(challengeReq).promise()
  
  console.log(`[${username}] - responded to auth challenge`)

  return {
    username,
    firstName,
    lastName,
    idToken: challengeResp.AuthenticationResult.IdToken
  }
}

module.exports = {
  an_authenticated_user
}
```

This requires the env vars `COGNITO_USER_POOL_ID` and `COGNITO_USER_POOL_SERVER_CLIENT_ID`.

These came from the `.env` file, via the CloudFormation outputs we configured in the `resouasrces.Outputs` section of the `serverless.yml`, which the `serverless-export-env` plugin has captured for us.

After each test, we also want to delete the test user so test data doesn't just accumulate in our environment.

4. Add a file `tearDown.js` to the `steps` folder and paste the following in

```javascript
const AWS = require('aws-sdk')

const an_authenticated_user = async (user) => {
  const cognito = new AWS.CognitoIdentityServiceProvider()
  
  let req = {
    UserPoolId: process.env.COGNITO_USER_POOL_ID,
    Username: user.username
  }
  await cognito.adminDeleteUser(req).promise()
  
  console.log(`[${user.username}] - user deleted`)
}

module.exports = {
  an_authenticated_user
}
```

5. Modify `steps/when.js` so that when we search restaurants, we would do so as an authenticated user.

Change the `we_invoke_search_restaurants` method to the following.

```javascript
const we_invoke_search_restaurants = async (theme, user) => {
  const body = JSON.stringify({ theme })

  switch (mode) {
    case 'handler':
      return await viaHandler({ body }, 'search-restaurants')
    case 'http':
      const auth = user.idToken
      return await viaHttp('restaurants/search', 'POST', { body, auth })
    default:
      throw new Error(`unsupported mode: ${mode}`)
  }
}
```

Notice that we're not taking in a `user` argument as well. This should be the authenticated Cognito user.

So let's go back to the test, and add steps to create and delete cognito users.

6. Open `test_cases/search-restaurants.tests.js` and replace the whole file with the following

```javascript
const { init } = require('../steps/init')
const when = require('../steps/when')
const tearDown = require('../steps/tearDown')
const given = require('../steps/given')
console.log = jest.fn()

describe('Given an authenticated user', () => {
  let user

  beforeAll(async () => {
    await init()
    user = await given.an_authenticated_user()
  })

  afterAll(async () => {
    await tearDown.an_authenticated_user(user)
  })

  describe(`When we invoke the POST /restaurants/search endpoint with theme 'cartoon'`, () => {
    it(`Should return an array of 4 restaurants`, async () => {
      const res = await when.we_invoke_search_restaurants('cartoon', user)

      expect(res.statusCode).toEqual(200)
      expect(res.body).toHaveLength(4)

      for (const restaurant of res.body) {
        expect(restaurant).toHaveProperty('name')
        expect(restaurant).toHaveProperty('image')
      }
    })
  })
})
```

7. Run the acceptance tests

`npm run acceptance`

and see that all 3 tests are passing

```
 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:39
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaur
ants

 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info tests/steps/when.js:39
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/

 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:39
      invoking via HTTP POST https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restau
rants/search


Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        4.098s
```

8. Run the integration tests and see that all 3 tests are still passing as well

```
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js

Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        4.393s
```

</p></details>

Congratulations, you now have both integration and acceptance tests!

![](/images/mod10-001.gif)