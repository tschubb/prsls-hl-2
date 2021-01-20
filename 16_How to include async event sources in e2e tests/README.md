# Module 16: How to include SNS and EventBridge in acceptance tests

## Validate messages published to SNS

**Goal:** Include SNS in the acceptance tests, so we can validate the message we publish to SNS

<details>
<summary><b>Add conditionally deployed SQS queue</b></summary><p>

1. Open `serverless.yml`.

2. Add the following `Conditions` block under the `resources` section

```yml
Conditions:
  IsE2eTest:
    Fn::Equals:
      - ${self:provider.stage}
      - dev
```

**IMPORTANT**: make sure that this section is aligned with `Resources` and `Outputs`. i.e.

```yml
resources:
  Conditions:
    ...

  Resources:
    ...

  Outputs:
    ...
```

We will use this `IsE2eTest` condition to conditionally deploy infrastructure resources for environments where we'll need to run end-to-end tests (which for now, is just the `dev` stage).

3. Add a SQS queue under `resources.Resources`

```yml
E2eTestQueue:
  Type: AWS::SQS::Queue
  Condition: IsE2eTest
  Properties:
    MessageRetentionPeriod: 60
    VisibilityTimeout: 1
```

Because this SQS queue is marked with the aforementioned `IsE2eTest` condition, it'll only be deployed (for now) when the `${self:provider.stage}` equals `dev`.

Notice that the `MessageRetentionPeriod` is set to 60s. This is because this queue is there primarily to facilitate end-to-end testing and doesn't need to retain messages beyond the duration of these tests. 1 minute is plenty of time for this use case.

Another thing to note is that `VisibilityTimeout` is set to a measely 1 second. Which means, means are available again after 1 second. This is partly necessary because Jest runs each test module in a separate environment, so messages that are picked up by one test would be temporarily hidden from another. Having a short visibility timeout should help with this as we increase the chance that each test would see each message at least once during the test.

4. To allow SNS to send messages to a SQS queue, we need to add a SQS queue policy and give `SQS:SendMessage` permission to the SNS topic.

Add the following to the `resources.Resources` section.

```yml
E2eTestQueuePolicy:
  Type: AWS::SQS::QueuePolicy
  Condition: IsE2eTest
  Properties:
    Queues:
      - !Ref E2eTestQueue
    PolicyDocument:
      Version: "2012-10-17"
      Statement:
        Effect: Allow
        Principal: "*"
        Action: SQS:SendMessage
        Resource: !GetAtt E2eTestQueue.Arn
        Condition:
          ArnEquals:
            aws:SourceArn: !Ref RestaurantNotificationTopic
```

Here you can see that, the `SQS:SendMessage` permission has been granted to the `RestaurantNotificationTopic` topic, and it's able to send messages to just the `E2eTestQueue` queue we configured in the previous step. So we're following security best practice and applying the principle of least privilege.

5. The last step to subscribe a SQS queue to receive messages from a SNS topic is to add an SNS subscription.

Add the following to the `resources.Resources` section.

```yml
E2eTestSnsSubscription:
  Type: AWS::SNS::Subscription
  Condition: IsE2eTest
  Properties:
    Protocol: sqs
    Endpoint: !GetAtt E2eTestQueue.Arn
    RawMessageDelivery: false
    Region: "#{AWS::Region}"
    TopicArn: !Ref RestaurantNotificationTopic
```

One thing that's worth pointing out here, is that `RawMessageDelivery` is set to `false`. This is an important detail.

If `RawMessageDelivery` is `true`, you will get just the message body that you publish to SNS as the SQS message body. For example:

```json
{
  "orderId": "4c67cf1d-9ac0-5dcb-9221-45726b7cbcc7",
  "restaurantName":"Pizza Planet"
}
```

Which is great when you just want to process the message. But it doesn't give us information about where the message came from, which is something that we need for our e2e tests, where we want to verify the right message was published to the right place.

With `RawMessageDelivery` set to `false`, this is what you receive in SQS instead:

```json
{
  "Type": "Notification",
  "MessageId": "8f14c0c1-6956-5fb7-a045-976ede2fe40b",
  "TopicArn": "arn:aws:sns:us-east-1:374852340823:workshop-yancui-dev-RestaurantNotificationTopic-1JUE46554XL3P",
  "Message": "{\"orderId\":\"4c67cf1d-9ac0-5dcb-9221-45726b7cbcc7\",\"restaurantName\":\"Pizza Planet\"}",
  "Timestamp": "2020-08-13T21:48:41.156Z",
  "SignatureVersion": "1",
  "Signature": "...",
  "SigningCertURL": "https://sns.us-east-1.amazonaws.com/...",
  "UnsubscribeURL": "https://sns.us-east-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=..."
}
```

From which we're able to identify where the message was sent from.

6. As good house-keeping and to allows us to reference it from the tests. Let's add the SQS queue's name to the stack outputs so it can be captured into our `.env` file.

Add the following to the `resources.Outputs` section.

```yml
E2eTestQueueUrl:
  Condition: IsE2eTest
  Value: !Ref E2eTestQueue
```

Notice that the `IsE2eTest` condition can be used on stack outputs too. If it's omitted here then the stack deployment would fail when the `IsE2etest` condition is false - because the resource `E2eTestQueue` wouldn't exist outside of the `dev` stack, and so this output would reference a non-existent resource.

7. Deploy the project.

`npx sls deploy`

This will provision a SQS queue and subscribe it to the SNS topic.

</p></details>

<details>
<summary><b>Check SNS messages in acceptance test</b></summary><p>

Now that we have added a SQS queue to catch all the messages that are published to SNS, let's integrate it into our acceptance test for the `notify-restaurant` function.

1. First, we need a way to trigger the `notify-restaurant` function in the end-to-end test. We can do this by publishing an event into the EventBridge bus.

Open `tests/steps/when.js`, and add this line to the top of the file

```js
const EventBridge = require('aws-sdk/clients/eventbridge')
```

and then add this method, right above the `viaHandler` method:

```js
const viaEventBridge = async (busName, source, detailType, detail) => {
  const eventBridge = new EventBridge()
  await eventBridge.putEvents({
    Entries: [{
      Source: source,
      DetailType: detailType,
      Detail: JSON.stringify(detail),
      EventBusName: busName
    }]
  }).promise()
}
```

2. Staying in `when.js`, replace `we_invoke_notify_restaurant` method with the following:

```js
const we_invoke_notify_restaurant = async (event) => {
  if (mode === 'handler') {
    await viaHandler(event, 'notify-restaurant')
  } else {
    const busName = process.env.EVENT_BUS_NAME
    await viaEventBridge(busName, event.source, event['detail-type'], event.detail)
  }
}
```

Here, we're using the new `viaEventBridge` method to trigger the deployed `notify-restaurant` function.

Next, we need a way to listen in on the messages that are captured in SQS.

3. Add a new file called `messages.js` under the `tests` folder.

4. Run `npm i --save-dev rxjs` to install RxJs, which has some really nice constructs for doing reactive programming in JavaScript.

5. Open the new `tests/messages.js` file you just added, and paste the following into it:

```js
const SQS = require('aws-sdk/clients/sqs')
const { ReplaySubject } = require("rxjs")
const {take, filter} = require("rxjs/operators")

const messages = new ReplaySubject(100)
const messageIds = new Set()
let pollingLoop

const startListening = () => {
  if (pollingLoop) {
    return
  }

  const sqs = new SQS()
  const queueUrl = process.env.E_2_E_TEST_QUEUE_URL
  const loop = async () => {
    const resp = await sqs.receiveMessage({
      QueueUrl: queueUrl,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20
    }).promise()

    if (!resp.Messages) {
      return await loop()
    }

    resp.Messages.forEach(msg => {
      if (messageIds.has(msg.MessageId)) {
        // seen this message already, ignore
        return
      }

      messageIds.add(msg.MessageId)

      const body = JSON.parse(msg.Body)
      if (body.TopicArn) {
        messages.next({
          sourceType: 'sns',
          source: body.TopicArn,
          message: body.Message
        })
      }
    })

    await loop()
  }

  pollingLoop = loop()
}

const waitForMessage = (sourceType, source, message) => {
  return messages
    .pipe(
        filter(incomingMessage => incomingMessage.sourceType === sourceType),
        filter(incomingMessage => incomingMessage.source === source),
        filter(incomingMessage => incomingMessage.message === message),
        take(1)
    ).toPromise()
}

module.exports = {
  startListening,
  waitForMessage
}
```

RxJs's [ReplaySubject](https://rxjs-dev.firebaseapp.com/api/index/class/ReplaySubject) lets you capture events and then replay them for every new subscriber. We will use it as a message buffer to capture all the messages that are in SQS, and when a test wants to wait for a specific message to arrive, we will replay through all the buffered messages.

When the test calls `startListening` we will use long-polling against SQS to pull in any messages it has:

```js
const resp = await sqs.receiveMessage({
  QueueUrl: queueUrl,
  MaxNumberOfMessages: 10,
  WaitTimeSeconds: 20
}).promise()
```

Because we disabled `RawMessageDelivery` in the SNS subscription, we have the necessary information to work out if a message has come from SNS topic. As you can see below, for each SQS message, we capture the SNS topic ARN as well as the actual message body.

```js
resp.Messages.forEach(msg => {
  const body = JSON.parse(msg.Body)
  if (body.TopicArn) {
    messages.next({
      sourceType: 'sns',
      source: body.TopicArn,
      message: body.Message
    })
  }
})
```

We do this on a loop, which does result in an annoying error message you will see later. It's possible to stop this but you'll have to implement cancellation for Promises ([bluebird](http://bluebirdjs.com/docs/getting-started.html) has a solution for this, if you want to explore this further).

The `waitForMessage` method is interesting in that it'll be called by different tests within a test module, each wanting to check if a different message has been received. This is where `ReplaySubject`'s ability to replay the same messages for every subscriber is crucial.

Also, other tests such as `place-order.tests.js` would also add messages onto the queue when it calls the `place-order` endpoint. Which is why when we wait for a message, we have to be thorough and look for exact messages.

So that's all the set up we need. Let's go and update the `notify-restaurant` function's test.

6. Open `tests/test_cases/notify-restaurant.tests.js` and replace it with the following

```js
const { init } = require('../steps/init')
const when = require('../steps/when')
const AWS = require('aws-sdk')
const chance = require('chance').Chance()
const messages = require('../messages')
console.log = jest.fn()

const mockPutEvents = jest.fn()
const mockPublish = jest.fn()

describe(`When we invoke the notify-restaurant function`, () => {
  const event = {
    source: 'big-mouth',
    'detail-type': 'order_placed',
    detail: {
      orderId: chance.guid(),
      restaurantName: 'Fangtasia'
    }
  }

  beforeAll(async () => {
    await init()

    if (process.env.TEST_MODE === 'handler') {
      AWS.EventBridge.prototype.putEvents = mockPutEvents
      AWS.SNS.prototype.publish = mockPublish

      mockPutEvents.mockReturnValue({
        promise: async () => {}
      })
      mockPublish.mockReturnValue({
        promise: async () => {}
      })
    } else {
      messages.startListening()
    }

    await when.we_invoke_notify_restaurant(event)
  })

  afterAll(() => {
    if (process.env.TEST_MODE === 'handler') {
      mockPutEvents.mockClear()
      mockPublish.mockClear()
    }
  })

  if (process.env.TEST_MODE === 'handler') {
    it(`Should publish message to SNS`, async () => {
      expect(mockPublish).toBeCalledWith({
        Message: expect.stringMatching(`"restaurantName":"Fangtasia"`),
        TopicArn: expect.stringMatching(process.env.restaurant_notification_topic)
      })
    })

    it(`Should publish event to EventBridge`, async () => {
      expect(mockPutEvents).toBeCalledWith({
        Entries: [
          expect.objectContaining({
            Source: 'big-mouth',
            DetailType: 'restaurant_notified',
            Detail: expect.stringContaining(`"restaurantName":"Fangtasia"`),
            EventBusName: expect.stringMatching(process.env.bus_name)
          })
        ]
      })
    })
  } else {
    it(`Should publish message to SNS`, async () => {
      await messages.waitForMessage(
        'sns',
        process.env.RESTAURANT_NOTIFICATION_TOPIC_ARN,
        JSON.stringify(event.detail)
      )
    }, 10000)
  }
})
```

Ok, a lot has changed in this file, let's walk through some of these changes.

In the `beforeAll`, the mocks are only configured when the `TEST_MODE` is `handler` - i.e. when we're running our integration tests by running the Lambda functions locally. **Otherwise, ask the aforementioned `messages` module to start listening for messages in the SQS queue**

```js
if (process.env.TEST_MODE === 'handler') {
  AWS.EventBridge.prototype.putEvents = mockPutEvents
  AWS.SNS.prototype.publish = mockPublish

  mockPutEvents.mockReturnValue({
    promise: async () => {}
  })
  mockPublish.mockReturnValue({
    promise: async () => {}
  })
} else {
  messages.startListening()
}
```

And since we don't have a way to capture EventBridge events yet, we are going to add a single test for now, to check that a message is published to SNS and that it's published to the right SNS topic and has the right payload.

```js
} else {
  it(`Should publish message to SNS`, async () => {
    await messages.waitForMessage(
      'sns',
      process.env.RESTAURANT_NOTIFICATION_TOPIC_ARN,
      JSON.stringify(event.detail)
    )
  }, 10000)
}
```

7. Run the integration test again

`npm run test`

everything should be passing.

```

 PASS  tests/test_cases/notify-restaurant.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/place-order.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info
      this is a secret

      at Function.module.exports.handler.middy (functions/search-restaurants.js:24:11)


Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        5.194 s
Ran all test suites.
```

8. Now run the acceptance tests

`npm run acceptance`

and now there's a new test for `notify-restaurant` and everything should be passing still

```
 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.info
      invoking via HTTP GET https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/restaurants

      at viaHttp (tests/steps/when.js:52:11)

 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info
      invoking via HTTP GET https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/

      at viaHttp (tests/steps/when.js:52:11)

 PASS  tests/test_cases/notify-restaurant.tests.js

ReferenceError: You are trying to `import` a file after the Jest environment has been torn down.

      at Object.userAgent (node_modules/aws-sdk/lib/util.js:34:43)
      at HttpRequest.setUserAgent (node_modules/aws-sdk/lib/http.js:111:78)
      at new HttpRequest (node_modules/aws-sdk/lib/http.js:104:10)
      at new Request (node_modules/aws-sdk/lib/request.js:328:24)
      at features.constructor.makeRequest (node_modules/aws-sdk/lib/service.js:202:19)
 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info
      invoking via HTTP POST https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/restaurants/search

      at viaHttp (tests/steps/when.js:52:11)

 PASS  tests/test_cases/place-order.tests.js
  ● Console

    console.info
      invoking via HTTP POST https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/orders

      at viaHttp (tests/steps/when.js:52:11)

A worker process has failed to exit gracefully and has been force exited. This is likely caused by tests leaking due to improper teardown. Try running with --runInBand --detectOpenHandles to find leaks.

Test Suites: 5 passed, 5 total
Tests:       5 passed, 5 total
Snapshots:   0 total
Time:        4.868 s, estimated 5 s
Ran all test suites.
```

As you can see the tests are passing, but the logs has a `ReferenceError` message. As mentioned earlier, this is due to the `tests/messages.js` module keeps polling SQS even when the test has finished. To address, you need some mechanism to cancel existing promises to stop the loop.

But, at least we are able to assert that the `notify-restaurant` function is actually sending notifications to the restaurant, so that's progress!

We can do more. Let's apply the same technique and check the events we publish to EventBridge as well.

</p></details>

**Goal:** Include EventBridge in the acceptance tests, so we can validate the events we publish to EventBridge

<details>
<summary><b>Add conditionally deployed EventBridge rule</b></summary><p>

To listen in on events going into an EventBridge bus, we need to first create a rule.
Similar to before, let's first add an EventBridge rule that's conditionally deployed when the stage is dev.

1. Add the following to `resources.Resources`:

```yml
E2eTestEventBridgeRule:
  Type: AWS::Events::Rule
  Condition: IsE2eTest
  Properties:
    EventBusName: !Ref EventBus
    EventPattern:
      source: ["big-mouth"]
    State: ENABLED
    Targets:
      - Arn: !GetAtt E2eTestQueue.Arn
        Id: e2eTestQueue
        InputTransformer:
          InputPathsMap:
            source: "$.source"
            detailType: "$.detail-type"
            detail: "$.detail"
          InputTemplate: >
            {
              "event": {
                "source": <source>,
                "detail-type": <detailType>,
                "detail": <detail>
              },
              "eventBusName": "#{EventBus}"
            }
```

As you can see, our rule would match any event where `source` is `big-mouth`, and it send the matched events to the `E2eTestQueue`. But what's this `InputTransformer`?

By Default, EventBridge would forward the matched events as they are. For example, a `restaurant_notified` event would normally look like this:

```json
{
  "version": "0",
  "id": "8520ecf2-f017-aec3-170d-6421916a5cf2",
  "detail-type": "restaurant_notified",
  "source": "big-mouth",
  "account": "374852340823",
  "time": "2020-08-14T01:38:27Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "orderId": "e249e6b2-cabe-5c4f-a5e9-5153cea847fe",
    "restaurantName": "Fangtasia"
  }
}
```

But as discussed previously, this doesn't allow us to capture information about the event bus. Luckily, EventBridge allows you to transform the matched event before sending them on to the target.

It does this in two steps:

Step 1 - use `InputPathsMap` to turn the event above into a property bag of key-value pairs. You can use the `$` symbol to navigate to the attributes you want - e.g. `$.detail`, or `$.detail.orderId`.

In our case, we want to capture the the `source`, `detail-type` and `detail`, which are the information that we sent from our code. And so our configuration below would map the matched event to 3 properties - `source`, `detailType` and `detail`.

```yml
InputPathsMap:
  source: "$.source"
  detailType: "$.detail-type"
  detail: "$.detail"
```

Step 2 - use `InputTemplate` to generate a string (doesn't have to be JSON). This template can reference properties we captured in Step 1 using the syntax `<PROPERTY_NAME>`.

In our case, I want to forward a JSON structure like this to SQS:

```json
{
  "event": {
    "source": "...",
    "detail-type": "...",
    "detail": {
      //...
    }
  },
  "eventBusName": "..."
}
```

Hence why I use the following template:

```yml
InputTemplate: >
  {
    "event": {
      "source": <source>,
      "detail-type": <detailType>,
      "detail": <detail>
    },
    "eventBusName": "#{EventBus}"
  }
```

ps. if you're not familiar with YML, the `>` symbol lets you insert a multi-line string. Read more about YML multi-line strings [here](https://yaml-multiline.info).

ps. if you recall, the `"#{EventBus}"` syntax is from the `serverless-pseudo-parameters` plugin. In this case, it's transformed this into a `Fn::Sub` in the generated CloudFormation stack:

```json
"InputTransformer": {
  "InputPathsMap": {
    "source": "$.source",
    "detailType": "$.detail-type",
    "detail": "$.detail"
  },
  "InputTemplate": {
    "Fn::Sub": "{\n  \"event\": {\n    \"source\": <source>,\n    \"detail-type\": <detailType>,\n    \"detail\": <detail>\n  },\n  \"eventBusName\": \"${EventBus}\"\n}\n"
  }
}
```

And references the `EventBus` resource - in this case, it's equivalent to `!Ref EventBus` but `Fn::Sub` allows you to do it inline. Have a look at `Fn::Sub`'s documentation page [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html) for more details.

Anyhow, with this `InputTransformer` configuration, this is how the events would look like in SQS:

```json
{
  "event": {
    "source": "big-mouth",
    "detail-type": "restaurant_notified",
    "detail": {
      "orderId": "e249e6b2-cabe-5c4f-a5e9-5153cea847fe",
      "restaurantName": "Fangtasia"
    }
  },
  "eventBusName": "order_events_dev"
}
```

2. We also need to give the EventBridge rule the necessary permission to push messages to `E2eTestQueue`. Luckily, we already have a `QueuePolicy` resource already, let's just update that.

Replace the `E2eTestQueuePolicy` resource in `resources.Resources` with the following:

```yml
E2eTestQueuePolicy:
  Type: AWS::SQS::QueuePolicy
  Condition: IsE2eTest
  Properties:
    Queues:
      - !Ref E2eTestQueue
    PolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal: "*"
          Action: SQS:SendMessage
          Resource: !GetAtt E2eTestQueue.Arn
          Condition:
            ArnEquals:
              aws:SourceArn: !Ref RestaurantNotificationTopic
        - Effect: Allow
          Principal: "*"
          Action: SQS:SendMessage
          Resource: !GetAtt E2eTestQueue.Arn
          Condition:
            ArnEquals:
              aws:SourceArn: !GetAtt E2eTestEventBridgeRule.Arn
```

Note that `Statement` can take a single item, or an array. So what we did here is to turn it into an array of statements, one to grant `SQS:SendMessage` permision to the `RestaurantNotificationTopic` topic and one for the `E2eTestEventBridgeRule` rule.

3. Redeploy the project

`npx sls deploy`

4. Go to the site, and place a few orders.

5. Go to the SQS console, and find your queue. You can see what messages are in the queue right here in the console.

Click `Send and receive messages`

![](/images/mod16-001.png)

In the following screen, click `Poll for messages`

![](/images/mod16-002.png)

You should see some messages come in. The smaller ones (~390 bytes) are EventBridge messages and the bigger ones (~1.07 kb) are SNS messages.

![](/images/mod16-003.png)

Click on them to see more details.

The SNS messages should look like this:

![](/images/mod16-004.png)

Whereas the EventBridge messages should look like this:

![](/images/mod16-005.png)

Ok, great, the EventBridge messages are captured in SQS, now we can add them to our tests.

</p></details>

<details>
<summary><b>Check EventBridge messages in acceptance test</b></summary><p>

We need to update the `tests/messages.js` module to capture messages from EventBridge too.

1. In `tests/messages.js`, on ln26, replace this block of code

```js
resp.Messages.forEach(msg => {
  if (messageIds.has(msg.MessageId)) {
    // seen this message already, ignore
    return
  }

  messageIds.add(msg.MessageId)

  const body = JSON.parse(msg.Body)
  if (body.TopicArn) {
    messages.next({
      sourceType: 'sns',
      source: body.TopicArn,
      message: body.Message
    })
  }
})
```

with the following:

```js
resp.Messages.forEach(msg => {
  if (messageIds.has(msg.MessageId)) {
    // seen this message already, ignore
    return
  }

  messageIds.add(msg.MessageId)

  const body = JSON.parse(msg.Body)
  if (body.TopicArn) {
    messages.next({
      sourceType: 'sns',
      source: body.TopicArn,
      message: body.Message
    })
  } else if (body.eventBusName) {
    messages.next({
      sourceType: 'eventbridge',
      source: body.eventBusName,
      message: JSON.stringify(body.event)
    })
  }
})
```

2. Go to `tests/test_cases/notify-restaurant.tests.js` and replace the whole file with the following:

```js
const { init } = require('../steps/init')
const when = require('../steps/when')
const chance = require('chance').Chance()
const messages = require('../messages')
console.log = jest.fn()

describe(`When we invoke the notify-restaurant function`, () => {
  const event = {
    source: 'big-mouth',
    'detail-type': 'order_placed',
    detail: {
      orderId: chance.guid(),
      restaurantName: 'Fangtasia'
    }
  }

  beforeAll(async () => {
    await init()
    messages.startListening()
    await when.we_invoke_notify_restaurant(event)
  })

  it(`Should publish message to SNS`, async () => {
    await messages.waitForMessage(
      'sns',
      process.env.RESTAURANT_NOTIFICATION_TOPIC_ARN,
      JSON.stringify(event.detail)
    )
  }, 10000)

  it(`Should publish "restaurant_notified" event to EventBridge`, async () => {
    await messages.waitForMessage(
      'eventbridge',
      process.env.EVENT_BUS_NAME,
      JSON.stringify({
        ...event,
        'detail-type': 'restaurant_notified'
      })
    )
  }, 10000)
})
```

Notice that we've done away with mocks altogether, and now our tests are simpler.

3. Run the integration tests

`npm run test`

and the new tests should pass (but you still see that annoying `ReferenceError` message in the logs)

```
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/notify-restaurant.tests.js

ReferenceError: You are trying to `import` a file after the Jest environment has been torn down.

      at Object.userAgent (node_modules/aws-sdk/lib/util.js:34:43)
      at HttpRequest.setUserAgent (node_modules/aws-sdk/lib/http.js:111:78)
      at new HttpRequest (node_modules/aws-sdk/lib/http.js:104:10)
      at new Request (node_modules/aws-sdk/lib/request.js:328:24)
      at features.constructor.makeRequest (node_modules/aws-sdk/lib/service.js:202:19)
 PASS  tests/test_cases/place-order.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info
      this is a secret

      at Function.module.exports.handler.middy (functions/search-restaurants.js:24:11)

A worker process has failed to exit gracefully and has been force exited. This is likely caused by tests leaking due to improper teardown. Try running with --runInBand --detectOpenHandles to find leaks.

Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        5.3 s
Ran all test suites.
```

4. Run the acceptance tests as well.

`npm run acceptance`

and they should be passing too.

```
 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.info
      invoking via HTTP GET https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/restaurants

      at viaHttp (tests/steps/when.js:52:11)

 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info
      invoking via HTTP GET https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/

      at viaHttp (tests/steps/when.js:52:11)

 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info
      invoking via HTTP POST https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/restaurants/search

      at viaHttp (tests/steps/when.js:52:11)

 PASS  tests/test_cases/place-order.tests.js
  ● Console

    console.info
      invoking via HTTP POST https://duiukrbz8l.execute-api.us-east-1.amazonaws.com/dev/orders

      at viaHttp (tests/steps/when.js:52:11)

 PASS  tests/test_cases/notify-restaurant.tests.js (5.287 s)

ReferenceError: You are trying to `import` a file after the Jest environment has been torn down.

      at Object.userAgent (node_modules/aws-sdk/lib/util.js:34:43)
      at HttpRequest.setUserAgent (node_modules/aws-sdk/lib/http.js:111:78)
      at new HttpRequest (node_modules/aws-sdk/lib/http.js:104:10)
      at new Request (node_modules/aws-sdk/lib/request.js:328:24)
      at features.constructor.makeRequest (node_modules/aws-sdk/lib/service.js:202:19)
A worker process has failed to exit gracefully and has been force exited. This is likely caused by tests leaking due to improper teardown. Try running with --runInBand --detectOpenHandles to find leaks.

Test Suites: 5 passed, 5 total
Tests:       6 passed, 6 total
Snapshots:   0 total
Time:        6.22 s, estimated 10 s
Ran all test suites.
```

Now let's do the same for the `place-order` function's test as well.

5. Open `tests/test_cases/place-order.tests.js` and replace the file with the following:

```js
const when = require('../steps/when')
const given = require('../steps/given')
const tearDown = require('../steps/tearDown')
const { init } = require('../steps/init')
const messages = require('../messages')
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

  describe(`When we invoke the POST /orders endpoint`, () => {
    let resp

    beforeAll(async () => {
      messages.startListening()
      resp = await when.we_invoke_place_order(user, 'Fangtasia')
    })

    it(`Should return 200`, async () => {
      expect(resp.statusCode).toEqual(200)
    })

    it(`Should publish a message to EventBridge bus`, async () => {
      const { orderId } = resp.body

      await messages.waitForMessage(
        'eventbridge',
        process.env.EVENT_BUS_NAME,
        JSON.stringify({
          source: 'big-mouth',
          'detail-type': 'order_placed',
          detail: {
            orderId,
            restaurantName: 'Fangtasia'
          }
        })
      )
    }, 10000)
  })
})
```

Again, no more mocks, we let our function talk to the real EventBridge bus and validate that the message was published correctly.

6. Rerun the integration tests

`npm run test`

7. Rerun the acceptance tests

`npm run acceptance`

And that's it, we are now validating the messages we publish to both SNS and EventBridge!

</p></details>