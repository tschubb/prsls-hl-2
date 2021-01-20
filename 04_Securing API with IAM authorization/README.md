# Module 4: Securing API with IAM authorization

## Securing get-restaurants endpoint with IAM-authorization

**Goal:** Protect the `get-restaurants` endpoint is with IAM

<details>
<summary><b>Use AWS_IAM authorizer for get-restaurants endpoint</b></summary><p>

1. Modify `serverless.yml` to add `authorizer: aws_iam` to the `get-restaurants` function

```yml
get-restaurants:
  handler: functions/get-restaurants.handler
  events:
    - http:
        path: /restaurants
        method: get
        authorizer: aws_iam
  environment:
    restaurants_table: !Ref RestaurantsTable
```
</p></details>

<details>
<summary><b>Add execute-api:Invoke to the IAM execution role</b></summary><p>

1. Modify `serverless.yml` to update the `iamRoleStatements` property

```yml
iamRoleStatements:
  - Effect: Allow
    Action: dynamodb:scan
    Resource: !GetAtt RestaurantsTable.Arn
  - Effect: Allow
    Action: execute-api:Invoke
    Resource: arn:aws:execute-api:#{AWS::Region}:#{AWS::AccountId}:#{ApiGatewayRestApi}/${self:provider.stage}/GET/restaurants
```

</p></details>

<details>
<summary><b>Signing request with IAM role</b></summary><p>

1. Install `aws4` as dependency

`npm install --save aws4`

This package lets us sign HTTP requests (with the [AWS v4 signing process](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html)) using our AWS credentials.

It applies the same logic as the AWS SDKs and looks for your AWS credentials in a number of places - in the environment variables, the AWS profiles (both `.aws/~config` and `.aws/~credentials`) using the EC2 instance metadata and ECS metadata, and so on.

2. Modify `get-index.js` to the following

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('axios')
const aws4 = require('aws4')
const URL = require('url')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']

const template = fs.readFileSync('static/index.html', 'utf-8')

const getRestaurants = async () => {
  console.log(`loading restaurants from ${restaurantsApiRoot}...`)
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname,
    path: url.pathname
  }

  aws4.sign(opts)

  const httpReq = http.get(restaurantsApiRoot, {
    headers: opts.headers
  })
  return (await httpReq).data
}

module.exports.handler = async (event, context) => {
  const restaurants = await getRestaurants()
  console.log(`found ${restaurants.length} restaurants`)  
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

3. Deploy the project

`npx sls deploy`

Reload the `index.html` and it should still work. But if you curl the `/restaurants` endpoint you should see

```
{
  message: "Missing Authentication Token"
}
```

</p></details>
