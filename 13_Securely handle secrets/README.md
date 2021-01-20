# Module 13: Securely handle secrets

## Store and load app secrets, securely

**Goal:** Load secrets from SSM Parameter Store with cache and cache invalidation

Following on from module 11, where we implemented mechanisms to load application configs at cold start and then cache and refresh every few minutes. However, we still end up storing the application configs in the environment variables. And it was OK for application configs, because they're not sensitive at all. Attackers are probably not very interested to know the default no. of restaurants you return in your homepage.

But when it comes to application secrets, we absolutely **SHOULD NOT** store them in the environment variables in plain text.

Instead, after we fetch (and decrypted) them during cold start, we want to set them on the invocation `context` object. Sure, it's not foolproof, and attackers can still find them if they know where to look. But they'll need to know an awful lot more about your application in order to do it. And in most cases, the attackers are not targetting us specifically, but we are caught in the net because we left low-hanging fruit for the attackers to pick. So, let's at least make them work for it!

<details>
<summary><b>Add a secure string to SSM Parameter Store</b></summary><p>

In our demo app, we don't actually have any secrets. But nonetheless, let's see how we *could* store and load these secrets from SSM Parameter Store and make sure that they're securely handled.

1. Go to `Systems Manager` console in AWS

2. Go to `Parameter Store`

3. Click `Create Parameter`

4. Use the name `/{service-name}/dev/search-restaurants/secretString` where `service-name` is the `service` name in your `serverless.yml`.

Choose `SecureString` as `Type`, and use the default `KMS` key.

For the value, put literally anything you want.

![](/images/mod13-001.png)

5. Click `Create Parameter`.

And now, we have a secret that is encrypted at rest, and whose access can be controlled via IAM. That's a pretty good start.

But we need to make sure when we distribute the secret to our application, we do so securely too.

</p></details>

<details>
<summary><b>Load secrets at runtime</b></summary><p>

1. Open `functions/search-restaurants.js`.

2. At the very end of the file, add the following.

```javascript
.use(ssm({
  cache: true,
  cacheExpiryInMillis: 5 * 60 * 1000, // 5 mins
  names: {
    secretString: `/${serviceName}/${stage}/search-restaurants/secretString`
  },
  setToContext: true,
  throwOnFailedCall: true
}))
```

After this change, the whole `module.exports.handler = ...` block of your function should look like this:

```javascript
module.exports.handler = middy(async (event, context) => {
  const req = JSON.parse(event.body)
  const theme = req.theme
  const restaurants = await findRestaurantsByTheme(theme, process.env.defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
}).use(ssm({
  cache: true,
  cacheExpiryInMillis: 5 * 60 * 1000, // 5 mins
  names: {
    config: `/${serviceName}/${stage}/search-restaurants/config`
  },
  onChange: () => {
    const config = JSON.parse(process.env.config)
    process.env.defaultResults = config.defaultResults
  }
})).use(ssm({
  cache: true,
  cacheExpiryInMillis: 5 * 60 * 1000, // 5 mins
  names: {
    secretString: `/${serviceName}/${stage}/search-restaurants/secretString`
  },
  setToContext: true,
  throwOnFailedCall: true
}))
```

So, let's talk about what's going on here.

Firstly, you can chain Middy middlewares by adding them one after another.

Secondly, unlike the application config, we asked the middleware to put the `secretString` in the `context` object instead of the environment variable.

If you want to see this working, then you can add a line such as

```javascript
console.info(context.secretString)
```

somewhere in the handler function, and run the integration test to see that it's printed in the terminal in plain text.

![](/images/mod13-002.png)

</p></details>

<details>
<summary><b>Configure IAM permissions</b></summary><p>

There's one last thing we need to do for this to work once we deploy the app - IAM permissions.

1. Open `serverless.yml`, and find the `iamRoleStatements` block under `provider`, add the following permission statement.

```yml
- Effect: Allow
  Action: ssm:GetParameters*
  Resource:
    - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/get-restaurants/config
    - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/search-restaurants/config
    - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/search-restaurants/secretString
```

After the change, the `iamRoleStatements` block should look like this.

```yml
iamRoleStatements:
  - Effect: Allow
    Action: dynamodb:scan
    Resource: !GetAtt RestaurantsTable.Arn
  - Effect: Allow
    Action: execute-api:Invoke
    Resource: arn:aws:execute-api:#{AWS::Region}:#{AWS::AccountId}:#{ApiGatewayRestApi}/${self:provider.stage}/GET/restaurants
  - Effect: Allow
    Action: ssm:GetParameters*
    Resource:
      - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/get-restaurants/config
      - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/search-restaurants/config
      - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/search-restaurants/secretString
```

2. Deploy the project

`npx sls deploy`

3. Run the acceptance to make sure everything is still working

`npm run acceptance`

```
  PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaurants

 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/

 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP POST https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaurant
s/search


Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        4.781s, estimated 5s
```

Notice that we didn't need to give our function `kms` permission to decrypt the `SecureString` parameter. This is because it was encrypted with an AWS managed key - `alias/aws/ssm` - which the SSM service has access to. So when we ask SSM for the parameter, it was able to decrypt it on our behalf.

This is no doubt a security concern. And if you want to tighten up the security of these secrets then you need to use a Customer Managed Key (CMK) - these are KMS keys that you create yourself and can control who have access to them.

4. Go to the `KMS` console in AWS, and click `Create Key`.

5. Follow through the instructions, and use the `service` name in your `serverless.yml` as alias for the key.

![](/images/mod13-003.png)

![](/images/mod13-004.png)

Choose who can administrator the key, for simplicity sake, choose the `Administrator` role and your current IAM user.

![](/images/mod13-005.png)

Choose who can use the key, in this case, add your IAM user.

![](/images/mod13-006.png)

Click `Finish` to create the key.

6. Go back to the `Systems Manager` console, and go to `Parameter Store`.

7. Find the `search-restaurants/secretString` parameter we created earlier, and click `Edit`.

Change the KMS key to the one we just created

![](/images/mod13-007.png)

Click `Save Changes`

8. Redeploy the project to force the code to reload the secret. Add the `--force` flag, otherwise Serverless framework might skip the deployment since we haven't changed anything.

`npx sls deploy --force`

9. Run the acceptance test again.

`npm run acceptance`

and see that the `search-restaurants` test is now failing.

```
 FAIL  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP POST https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaurant
s/search

  ● Given an authenticated user › When we invoke the POST /restaurants/search endpoint with theme 
'cartoon' › Should return an array of 4 restaurants

    Request failed with status code 502

      at createError (node_modules/axios/lib/core/createError.js:16:15)
      at settle (node_modules/axios/lib/core/settle.js:17:12)
      at IncomingMessage.handleStreamEnd (node_modules/axios/lib/adapters/http.js:236:11)

Test Suites: 1 failed, 2 passed, 3 total
Tests:       1 failed, 2 passed, 3 total
Snapshots:   0 total
Time:        3.768s, estimated 5s
```

10. Check the logs for this function

`npx sls logs -f search-restaurants`

and see the `AccessDeniedException`

![](/images/mod13-008.png)

That's good. Now, only those who was access to the KMS key would be able to access the secret. It's another layer of protection.

</p></details>

<details>
<summary><b>Configure IAM permissions for KMS</b></summary><p>

To give our function permission to decrypt the secret value using KMS, we need the ARN for the key. Although the AWS documentations seem to suggest that you can grant IAM permissions to CMK keys using alias, they have never worked for me.

So, as a result, the approach I normally take is for the process that provision these keys (in practice, we wouldn't be doing it by hand!) to also provision a SSM parameter with the key's ARN.

Since we don't have another project that manages these shared resources in the region (often, they're part of an organization's landing zone configuration), let's add this SSM parameter by hand.

1. Go to the `Parameter Store` console in `Systems Manager`.

2. Create a new `String` parameter called `/dev/kmsArn`, and put the ARN of the KMS key (you can find this in the `KMS` console if you navigate to the key you created earlier) as its value.

![](/images/mod13-009.png)

3. Open `serverless.yml`.

4. In the `iamRoleStatements` block, add the following permissions

```yml
- Effect: Allow
  Action: kms:Decrypt
  Resource: ${ssm:/dev/kmsArn}
```

This special syntax `${ssm:...}` is how we can reference parameters in SSM directly in our `serverless.yml`. It's useful for referencing things like this, but again, since the SSM parameter values are fetched at deployment time and baked into the generated CloudFormation template, you shouldn't load any secrets this way.

5. Redeploy the project

`npx sls deploy`

6. Rerun the acceptance test

`npm run acceptance`

and now everything passes again!

```
 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaurants

 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/

 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP POST https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaurant
s/search


Test Suites: 3 passed, 3 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        4.997s
```

</p></details>

<details>
<summary><b>Wrapping the wrapper</b></summary><p>

Looking at the `search-restaurants` function, it's getting a bit unsightly. I mean, half of it is to do with middlewares!

In that case, you can encapsulate some of these into your own wrapper so it's all tucked away.

e.g. if there are a number of middlewares that you always want to apply, you might create a wrapper like this.

```javascript
module.exports = f => {
  return middy(f)
    .use(middleware1({ ... }))
    .use(middleware2({ ... }))
    .use(middleware3({ ... }))
}
```

and then apply them to your function handlers like this.

```javascript
const wrap = require('../lib/wrapper')
...
module.exports.handler = wrap(async (event, context) => {
  ...
})
```

You may even do yourself a favour and `promisify` the callback-style function that *Middy* returns. e.g.

```javascript
const { promisify } = require('util')
...
module.exports = f => {
  const g = middy(f)
    .use(middleware1({ ... }))
    .use(middleware2({ ... }))
    .use(middleware3({ ... }))

  return promisify(g)
}
```

And if you have conventions regarding application configs or secrets, you can also encode those into your wrappers.

However, a word of caution here, I have seen many teams go overboard with this and create wrappers that are far too rigid and make too much assumptions (e.g. every function must have an application config in SSM).

So my advice is to apply this technique with a sense of reserved caution, and only bundle in middlewares that you know are required for EVERYONE.

Another thing I'd advice against, is putting business logic into middlewares. Again, I've seen far too many teams go trigger-happy with middlewares and start putting everything in them. Middlwares are powerful tools but they're just tools nonetheless. Master your tools, and don't let them master you.

</p></details>
