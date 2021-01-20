# Module 18: Per function IAM roles

## Give each function a dedicated IAM role

**Goal:** Each function has a dedicated IAM role to minimise attack surface

<details>
<summary><b>Install serverless-iam-roles-per-function plugin</b></summary><p>

1. Install `serverless-iam-roles-per-function` as dev dependency

`npm install --save-dev serverless-iam-roles-per-function@next`

2. Modify `serverless.yml` and add it as a plugin

```yml
plugins:
  - serverless-export-env
  - serverless-pseudo-parameters
  - serverless-iam-roles-per-function
```

</p></details>

<details>
<summary><b>Issue individual permissions</b></summary><p>

1. Open `serverless.yml` and delete the `iamRoleStatements` section

2. Give the `get-index` function its own IAM role statements by adding the following to its definition

```yml
iamRoleStatements:
  - Effect: Allow
    Action: execute-api:Invoke
    Resource: arn:aws:execute-api:#{AWS::Region}:#{AWS::AccountId}:#{ApiGatewayRestApi}/${self:provider.stage}/GET/restaurants
```

**IMPORTANT** this new block should be aligned with `environment` and `events`, e.g.

```yml
get-index:
  handler: functions/get-index.handler
  events: ...
  environment:
    restaurants_api: ...
    orders_api: ...
    cognito_user_pool_id: ...
    cognito_client_id: ...
  iamRoleStatements:
    - Effect: Allow
      Action: execute-api:Invoke
      Resource: arn:aws:execute-api:#{AWS::Region}:#{AWS::AccountId}:#{ApiGatewayRestApi}/${self:provider.stage}/GET/restaurants
```

3. Similarly, give the `get-restaurants` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: dynamodb:scan
    Resource: !GetAtt RestaurantsTable.Arn
  - Effect: Allow
    Action: ssm:GetParameters*
    Resource: arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/get-restaurants/config
```

4. Give the `search-restaurants` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: dynamodb:scan
    Resource: !GetAtt RestaurantsTable.Arn
  - Effect: Allow
    Action: ssm:GetParameters*
    Resource:
      - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/search-restaurants/config
      - arn:aws:ssm:#{AWS::Region}:#{AWS::AccountId}:parameter/${self:service}/${self:provider.stage}/search-restaurants/secretString
  - Effect: Allow
    Action: kms:Decrypt
    Resource: ${ssm:/dev/kmsArn}
```

5. Give the `place-order` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: events:PutEvents
    Resource: !GetAtt EventBus.Arn
```

6. Finally, give the `notify-restaurant` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: events:PutEvents
    Resource: !GetAtt EventBus.Arn
  - Effect: Allow
    Action: sns:Publish
    Resource: !Ref RestaurantNotificationTopic
```

7. Deploy the project

`npx sls deploy`

8. Run the acceptance tests to make sure they're still working

`npm run acceptance`

But, since we don't have acceptance test coverage for the `notify-restaurant` function, we need to manually verify it's still working.

9. Use the `lumigo-cli` to listen to the `EventBridge` event bus and the `SNS` topic. You need to run the [tail-eventbridge-bus](https://www.npmjs.com/package/lumigo-cli#lumigo-cli-tail-eventbridge-bus) and [tail-sns](https://www.npmjs.com/package/lumigo-cli#lumigo-cli-tail-sns) commands respectively.

**Hint**: you can find the event bus name and SNS topic name in the `.env` file.

10. Load the index page and place an order by clicking on one of the restaurants. See that the events are captured in the `EventBridge` bus and the `SNS` was published to the topic.

</p></details>
