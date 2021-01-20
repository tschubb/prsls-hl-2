# Module 15: Processing events with EventBridge

## Process events in realtime with EventBridge and Lambda

**Goal:** Implemented the order flow using EventBridge

<details>
<summary><b>Add EventBridge bus</b></summary><p>

1. Open `serverless.yml`.

2. Add an `EventBridge` bus as a new resource under the `resources.Resources` section

```yml
EventBus:
  Type: AWS::Events::EventBus
  Properties:
    Name: order_events_${self:provider.stage}
```

**IMPORTANT**: make sure that this `EventBus` resource is aligned with `ServiceUrlParameter`, `CognitoAuthorizer` and other CloudFormation resources.

3. While we're here, let's also add the EventBus name as output. Add the following to the `resources.Outputs` section.

```yml
EventBusName:
  Value: !Ref EventBus
```

4. Deploy the project.

`npx sls deploy`

This will provision an EventBridge bus called `order_events_dev`.

</p></details>

<details>
<summary><b>Add place-order function</b></summary><p>

1. Add a new `place-order` function (in the `functions` section)

```yml
place-order:
  handler: functions/place-order.handler
  events:
    - http:
        path: /orders
        method: post
        authorizer:
          name: CognitoAuthorizer
          type: COGNITO_USER_POOLS
          arn: !GetAtt CognitoUserPool.Arn
  environment:
    bus_name: !Ref EventBus
```

Notice that this new function references the newly created `EventBridge` bus, whose name will be passed in via the `bus_name` environment variable.

This function also uses the same Cognito User Tool for authorization, as it'll be called directly by the client app.

2. Add the permission to publish events to `EventBridge` by adding the following to the list of permissions under `iamRoleStatements`:

```yml
- Effect: Allow
  Action: events:PutEvents
  Resource: !GetAtt EventBus.Arn
```

3. Add a file `place-order.js` to the `functions` folder

4. Modify `place-order.js` to the following

```javascript
const EventBridge = require('aws-sdk/clients/eventbridge')
const eventBridge = new EventBridge()
const chance = require('chance').Chance()

const busName = process.env.bus_name

module.exports.handler = async (event) => {
  const restaurantName = JSON.parse(event.body).restaurantName

  const orderId = chance.guid()
  console.log(`placing order ID [${orderId}] to [${restaurantName}]`)

  await eventBridge.putEvents({
    Entries: [{
      Source: 'big-mouth',
      DetailType: 'order_placed',
      Detail: JSON.stringify({
        orderId,
        restaurantName,
      }),
      EventBusName: busName
    }]
  }).promise()

  console.log(`published 'order_placed' event into EventBridge`)

  const response = {
    statusCode: 200,
    body: JSON.stringify({ orderId })
  }

  return response
}
```

This `place-order` function handles requests to create an order (via the `POST /orders` endpoint we configured just now).

As part of the `POST` body in the request, it expects the `restaurantName` to be passed in. And upon receiving a request, all it's doing is publishing an event to the `EventBridge` bus and let some other process handle it.

</p></details>

<details>
<summary><b>Add integration test for place-order function</b></summary><p>

1. Add a file `place-order.tests.js` to `test_cases` folder

2. Modify `test_cases/place-order.tests.js` to the following

```javascript
const when = require('../steps/when')
const given = require('../steps/given')
const tearDown = require('../steps/tearDown')
const { init } = require('../steps/init')
const AWS = require('aws-sdk')
console.log = jest.fn()

const mockPutEvents = jest.fn()
AWS.EventBridge.prototype.putEvents = mockPutEvents

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
      mockPutEvents.mockClear()
      mockPutEvents.mockReturnValue({
        promise: async () => {}
      })

      resp = await when.we_invoke_place_order(user, 'Fangtasia')
    })

    it(`Should return 200`, async () => {
      expect(resp.statusCode).toEqual(200)
    })

    it(`Should publish a message to EventBridge bus`, async () => {
      expect(mockPutEvents).toBeCalledWith({
        Entries: [
          expect.objectContaining({
            Source: 'big-mouth',
            DetailType: 'order_placed',
            Detail: expect.stringContaining(`"restaurantName":"Fangtasia"`),
            EventBusName: expect.stringMatching(process.env.bus_name)
          })
        ]
      })
    })
  })
})
```

Wait a minute, we're mocking the AWS operations! Didn't you say not to do it?

Yes, I did...

The problem is that, to validate the events that are sent to `EventBridge` it'll take a bit of extra infrastructure set up. Because you can't just call `EventBridge` and ask what events it had just received on a bus recently. You need to subscribe to the bus and capture events in real-time as they happen.

We'll explore how to do this in the next couple of modules. For now, let's just mock these tests.

3. Modify `steps/when.js` to add a new `we_invoke_place_order` function

```javascript
const we_invoke_place_order = async (user, restaurantName) => {
  const body = JSON.stringify({ restaurantName })

  switch (mode) {
    case 'handler':
      return await viaHandler({ body }, 'place-order')
    case 'http':
      const auth = user.idToken
      return await viaHttp('orders', 'POST', { body, auth })
    default:
      throw new Error(`unsupported mode: ${mode}`)
  }
}
```

and don't forget to add it to the list of exported methods too

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order
}
```

4. Run integration tests

`npm run test`

and see that all 5 tests are passing

```
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/place-order.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js (5.041s)
  ● Console

    console.info functions/search-restaurants.js:24
      this is a new secret


Test Suites: 4 passed, 4 total
Tests:       5 passed, 5 total
Snapshots:   0 total
Time:        5.431s
```

5. Deploy the project

`npx sls deploy`

</p></details>

<details>
<summary><b>Add acceptance test for place-order function</b></summary><p>

When executing the deployed `place-order` function via API Gateway, the function would publish an `order_placed` event to the real EventBridge bus.

To verify that the event is published as expected, you have some options (discussed in [this post](https://theburningmonk.com/2019/09/how-to-include-sns-and-kinesis-in-your-e2e-tests/)). Again, for the purpose of this workshop, we'll take a short-cut and only validate EventBridge was called when executing as an integration test, using mocks...

1. Modify `test_cases/place-order.tests.js` so the `Should publish a message to EventBridge bus` test case only runs as an integration test.

Wrap the test case

```javascript
it(`Should publish a message to EventBridge bus`, async () => {
  expect(mockPutEvents).toBeCalledWith({
    Entries: [
      expect.objectContaining({
        Source: 'big-mouth',
        DetailType: 'order_placed',
        Detail: expect.stringContaining(`"restaurantName":"Fangtasia"`),
        EventBusName: expect.stringMatching(process.env.bus_name)
      })
    ]
  })
})
```

in an `if` block like this

```javascript
if (process.env.TEST_MODE === 'handler') {
  it(`Should publish a message to EventBridge bus`, async () => {
    expect(mockPutEvents).toBeCalledWith({
      Entries: [
        expect.objectContaining({
          Source: 'big-mouth',
          DetailType: 'order_placed',
          Detail: expect.stringContaining(`"restaurantName":"Fangtasia"`),
          EventBusName: expect.stringMatching(process.env.bus_name)
        })
      ]
    })
  })
}
```

2. Run acceptance test

`npm run acceptance`

and see that we have 4 (instead of 5 for integration) tests, and they're all passing.

```
 PASS  tests/test_cases/get-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaurants

 PASS  tests/test_cases/get-index.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP GET https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/

 PASS  tests/test_cases/place-order.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP POST https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/orders

 PASS  tests/test_cases/search-restaurants.tests.js
  ● Console

    console.info tests/steps/when.js:40
      invoking via HTTP POST https://4q8sbvheq2.execute-api.us-east-1.amazonaws.com/dev/restaurant
s/search


Test Suites: 4 passed, 4 total
Tests:       4 passed, 4 total
Snapshots:   0 total
Time:        4.845s
```

Again, we'll see how we can extend these acceptance tests to validate the events that are published to EventBridge and SNS.

</p></details>

<details>
<summary><b>Update web client to support placing order</b></summary><p>

1. Modify `static/index.html` to the following

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>

    <script src="https://sdk.amazonaws.com/js/aws-sdk-2.149.0.min.js"></script>
    <script src="https://d2qt42rcwzspd6.cloudfront.net/manning/aws-cognito-sdk.min.js"></script>
    <script src="https://d2qt42rcwzspd6.cloudfront.net/manning/amazon-cognito-identity.min.js"></script>
    <script src="https://code.jquery.com/jquery-3.2.1.min.js" 
            integrity="sha256-hwg4gsxgFZhOsEEamdOYGBf13FyQuiTwlAQgxVSNgt4="
            crossorigin="anonymous"></script>
    <script src="https://code.jquery.com/ui/1.12.1/jquery-ui.min.js" 
            integrity="sha384-Dziy8F2VlJQLMShA6FHWNul/veM9bCkRUaLqr199K94ntO5QUrLJBEbYegdSkkqX" 
            crossorigin="anonymous"></script>
    <link rel="stylesheet" href="https://code.jquery.com/ui/1.12.1/themes/base/jquery-ui.css">

    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;        
      }
      .restaurantsDiv {
        background-color: #ffffff;
        width: 100%;
        height: auto;
      }
      .dayOfWeek {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 32px;
        padding: 10px;
        height: auto;
        display: flex;
        justify-content: center;
      }
      .column-container {
        padding: 0;
        margin: 0;        
        list-style: none;
        display: flex;
        flex-flow: column;
        flex-wrap: wrap;
        justify-content: center;
      }
      .row-container {
        padding: 5px;
        margin: 5px;
        list-style: none;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .restaurant {
        background-color: #00a8f7;
        border-radius: 10px;
        padding: 5px;
        height: auto;
        width: auto;
        margin-left: 40px;
        margin-right: 40px;
        margin-top: 15px;
        margin-bottom: 0px;
        display: flex;
        justify-content: center;
      }
      .restaurant-name {
        font-size: 24px;
        font-family:Arial, Helvetica, sans-serif;
        color: #ffffff;
        padding: 10px;
        margin: 0px;
      }
      .restaurant-image {
        padding-top: 0px;
        margin-top: 0px;
      }
      .row-container-left {
        list-style: none;
        display: flex;
        flex-flow: row;
        justify-content: flex-start;
      }
      .menu-text {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 24px;
        font-weight: bold;
        color: white;
      }
      .text-trail-space {
        margin-right: 10px;
      }
      .hidden {
        display: none;
      }

      label, button, input {
        display:block;
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
      
      fieldset { 
        padding:0; 
        border:0; 
        margin-top:25px; 
      }

    </style>

    <script>
      const AWS_REGION = '{{awsRegion}}';
      const COGNITO_USER_POOL_ID = '{{cognitoUserPoolId}}';
      const CLIENT_ID = '{{cognitoClientId}}';
      const SEARCH_URL = '{{& searchUrl}}';
      const PLACE_ORDER_URL = '{{& placeOrderUrl}}';

      var regDialog, regForm;
      var verifyDialog;
      var regCompleteDialog;
      var signInDialog;
      var userPool, cognitoUser;
      var idToken;

      function toggleSignOut (enable) {
        enable === true ? $('#sign-out').show() : $('#sign-out').hide();
      }

      function toggleSignIn (enable) {
        enable === true ? $('#sign-in').show() : $('#sign-in').hide();
      }

      function toggleRegister (enable) {
        enable === true ? $('#register').show() : $('#register').hide();
      }

      function init() {
        AWS.config.region = AWS_REGION;
        AWSCognito.config.region = AWS_REGION;

        var data = { 
          UserPoolId : COGNITO_USER_POOL_ID, 
          ClientId : CLIENT_ID
        };
        userPool = new AWSCognito.CognitoIdentityServiceProvider.CognitoUserPool(data);
        cognitoUser = userPool.getCurrentUser();

        if (cognitoUser != null) {          
          cognitoUser.getSession(function(err, session) {
            if (err) {
                alert(err);
                return;
            }

            idToken = session.idToken.jwtToken;
            console.log('idToken: ' + idToken);
            console.log('session validity: ' + session.isValid());
          });

          toggleSignOut(true);
          toggleSignIn(false);
          toggleRegister(false);
        } else {
          toggleSignOut(false);
          toggleSignIn(true);
          toggleRegister(true);
        }
      }

      function addUser() {
        var firstName = $("#first-name")[0].value;
        var lastName = $("#last-name")[0].value;
        var username = $("#username")[0].value;
        var password = $("#password")[0].value;
        var email = $("#email")[0].value;

        var attributeList = [
          new AWSCognito.CognitoIdentityServiceProvider.CognitoUserAttribute({ 
            Name : 'email', Value : email
          }),
          new AWSCognito.CognitoIdentityServiceProvider.CognitoUserAttribute({ 
            Name : 'given_name', Value : firstName
          }),
          new AWSCognito.CognitoIdentityServiceProvider.CognitoUserAttribute({ 
            Name : 'family_name', Value : lastName
          }),
        ];

        userPool.signUp(username, password, attributeList, null, function(err, result){
          if (err) {
            alert(err);
            return;
          }
          cognitoUser = result.user;
          console.log('user name is ' + cognitoUser.getUsername());

          regDialog.dialog("close");
          verifyDialog.dialog("open");
        });
      }

      function confirmUser() {
        var verificationCode = $("#verification-code")[0].value;
        cognitoUser.confirmRegistration(verificationCode, true, function(err, result) {
          if (err) {
            alert(err);
            return;
          }
          console.log('verification call result: ' + result);

          verifyDialog.dialog("close");
          regCompleteDialog.dialog("open");
        });
      }

      function authenticateUser() {
        var username = $("#sign-in-username")[0].value;
        var password = $("#sign-in-password")[0].value;

        var authenticationData = {
          Username : username,
          Password : password,
        };
        var authenticationDetails = new AWSCognito.CognitoIdentityServiceProvider.AuthenticationDetails(authenticationData);
        var userData = {
          Username : username,
          Pool : userPool
        };
        var cognitoUser = new AWSCognito.CognitoIdentityServiceProvider.CognitoUser(userData);

        cognitoUser.authenticateUser(authenticationDetails, {
          onSuccess: function (result) {
            console.log('access token : ' + result.getAccessToken().getJwtToken());
            /*Use the idToken for Logins Map when Federating User Pools with Cognito Identity or when passing through an Authorization Header to an API Gateway Authorizer*/
            idToken = result.idToken.jwtToken;
            console.log('idToken : ' + idToken);

            signInDialog.dialog("close");
            toggleRegister(false);
            toggleSignIn(false);
            toggleSignOut(true);
          },

          onFailure: function(err) {
            alert(err);
          }
        });
      }

      function signOut() {
        if (cognitoUser != null) {
          cognitoUser.signOut();
          toggleRegister(true);
          toggleSignIn(true);
          toggleSignOut(false);
        }
      }

      function searchRestaurants() {
        var theme = $("#theme")[0].value;

        var xhr = new XMLHttpRequest();
        xhr.open('POST', SEARCH_URL, true);
        xhr.setRequestHeader("Content-Type", "application/json");
        xhr.setRequestHeader("Authorization", idToken);
        xhr.send(JSON.stringify({ theme }));
        
        xhr.onreadystatechange = function (e) {
          if (xhr.readyState === 4 && xhr.status === 200) {
            var restaurants = JSON.parse(xhr.responseText);
            var restaurantsList = $("#restaurantsUl");
            restaurantsList.empty();

            for (var restaurant of restaurants) {
              restaurantsList.append(`
              <li class="restaurant">
                <ul class="column-container" onclick='placeOrder("${restaurant.name}")'>
                    <li class="item restaurant-name">${restaurant.name}</li>
                    <li class="item restaurant-image">
                      <img src="${restaurant.image}">
                    </li>
                </ul>
              </li>
              `);
            }

          } else if (xhr.readyState === 4) {
            alert(xhr.responseText);
          }
        };
      }

      function placeOrder(restaurantName) {
        var xhr = new XMLHttpRequest();
        xhr.open('POST', PLACE_ORDER_URL, true);
        xhr.setRequestHeader("Content-Type", "application/json");
        xhr.setRequestHeader("Authorization", idToken);
        xhr.send(JSON.stringify({ restaurantName }));

        xhr.onreadystatechange = function (e) {
          if (xhr.readyState === 4 && xhr.status === 200) {
            alert("your order has been placed, we'll let you know once it's been accepted by the restaurant!");
          } else if (xhr.readyState === 4) {
            alert(xhr.responseText);
          }
        };
      }

      $(document).ready(function() {
        regDialog = $("#reg-dialog-form").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            "Create an account": addUser,
            Cancel: function() {
              regDialog.dialog("close");
            }
          },
          close: function() {
            regForm[0].reset();
          }
        });

        regForm = regDialog.find("form").on("submit", function(event) {
          event.preventDefault();
          addUser();
        });
        
        $("#register").on("click", function() {
          regDialog.dialog("open");
        });

        verifyDialog = $("#verify-dialog-form").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            "Confirm registration": confirmUser,
            Cancel: function() {
              verifyDialog.dialog("close");
            }
          },
          close: function() {
            $(this).dialog("close");
          }
        });

        regCompleteDialog = $("#registered-message").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            Ok: function() {
              $(this).dialog("close");
            }
          }
        });

        signInDialog = $("#sign-in-form").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            "Sign in": authenticateUser,
            Cancel: function() {
              signInDialog.dialog("close");
            }
          },
          close: function() {
            $(this).dialog("close");
          }
        });

        $("#sign-in").on("click", function() {
          signInDialog.dialog("open");
        });

        $("#sign-out").on("click", function() {
          signOut();
        })

        init();
      });

    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li>
          <ul class="row-container-left">
            <li id="register" class="item text-trail-space hidden">
              <a class="menu-text" href="#">Register</a>
            </li>
            <li id="sign-in" class="item menu-text text-trail-space hidden">
              <a class="menu-text" href="#">Sign in</a>
            </li>
            <li id="sign-out" class="item menu-text text-trail-space hidden">
              <a class="menu-text" href="#">Sign out</a>
            </li>
          </ul>
        </li>
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. rick and morty"/>
          <button onclick="searchRestaurants()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul id="restaurantsUl" class="row-container">
              {{#restaurants}}
              <li class="restaurant">
                <ul class="column-container" onclick='placeOrder("{{name}}")'>
                    <li class="item restaurant-name">{{name}}</li>
                    <li class="item restaurant-image">
                      <img src="{{image}}">
                    </li>
                </ul>
              </li>
              {{/restaurants}}
            </ul>
          </div>
        </li>
      </ul>
    </div>

    <div id="reg-dialog-form" title="Register">       
      <form>
        <fieldset>
          <label for="first-name">First Name</label>
          <input type="text" id="first-name" class="text ui-widget-content ui-corner-all">
          <label for="last-name">Last Name</label>
          <input type="text" id="last-name" class="text ui-widget-content ui-corner-all">
          <label for="email">Email</label>
          <input type="text" name="email" id="email" class="text ui-widget-content ui-corner-all">
          <label for="username">Username</label>
          <input type="text" name="username" id="username" class="text ui-widget-content ui-corner-all">
          <label for="password">Password</label>
          <input type="password" name="password" id="password" class="text ui-widget-content ui-corner-all">
        </fieldset>
      </form>
    </div>

    <div id="verify-dialog-form" title="Verify">
      <form>
        <fieldset>
            <label for="verification-code">Verification Code</label>
            <input type="text" id="verification-code" class="text ui-widget-content ui-corner-all">
        </fieldset>
      </form>
    </div>

    <div id="registered-message" title="Registration complete!">
      <p>
        <span class="ui-icon ui-icon-circle-check" style="float:left; margin:0 7px 50px 0;"></span>
        You are now registered!
      </p>
    </div>

    <div id="sign-in-form" title="Sign in">
      <form>
          <fieldset>            
            <label for="sign-in-username">Username</label>
            <input type="text" id="sign-in-username" class="text ui-widget-content ui-corner-all">
            <label for="sign-in-password">Password</label>
            <input type="password" id="sign-in-password" class="text ui-widget-content ui-corner-all">
          </fieldset>
        </form>
    </div>

  </body>

</html>
```

This new UI code would call the `POST /orders` endpoint when you click on one of the restaurants.

But to do that, the `get-index` function needs to know the URL endpoint for it, and then pass it into the HTML template.

2. Open `serverless.yml` and add an `orders_api` environment variable to the `get-index` function.

```yml
orders_api:
  Fn::Join:
    - ""
    - - https://
      - !Ref ApiGatewayRestApi
      - .execute-api.${self:provider.region}.amazonaws.com/${self:provider.stage}/orders
```

3. Modify `functions/get-index.js` to fetch the URL endpoint to place orders (from a new `orders_api` environment variable). On ln8 where you have:

```javascript
const restaurantsApiRoot = process.env.restaurants_api
```

Somewhere near there, add the following:

```javascript
const ordersApiRoot = process.env.orders_api
```

4. Modify `functions/get-index.js` to pass the `ordersApiRoot` url to the updated `index.html` template. On ln38, replace the `view` object so we add a `placeOrderUrl` field.

```javascript
const view = {
  awsRegion,
  cognitoUserPoolId,
  cognitoClientId,
  dayOfWeek,
  restaurants,
  searchUrl: `${restaurantsApiRoot}/search`,
  placeOrderUrl: `${ordersApiRoot}`
}
```

5. Deploy the project

`npx sls deploy`

Load the landing page in the browser and click on one of the restaurants to order (if your login token has expired then you'll have to sign in again)

![](/images/mod15-001.png)

</p></details>

<details>
<summary><b>Add notify-restaurant function</b></summary><p>

1. Modify `serverless.yml` to add a new SNS topic for notifying restaurants, under the `resources.Resources` section

```yml
RestaurantNotificationTopic:
  Type: AWS::SNS::Topic
```

**IMPORTANT**: make sure this is aligned with other CloudFormation resources, like the `EventBus` resoure we added earlier.

2. Also, add the SNS topic's name and ARN to our stack output. Add the following to the `resources.Outputs` section of the `serverless.yml`.

```yml
RestaurantNotificationTopicName:
  Value: !GetAtt RestaurantNotificationTopic.TopicName

RestaurantNotificationTopicArn:
  Value: !Ref RestaurantNotificationTopic
```

3. Deploy the project to provision the SNS topic.

`npx sls deploy`

4. Add a file `notify-restaurant.js` in the `functions` folder

5. Modify `functions/notify-restaurant.js` to the following

```javascript
const EventBridge = require('aws-sdk/clients/eventbridge')
const eventBridge = new EventBridge()
const SNS = require('aws-sdk/clients/sns')
const sns = new SNS()

const busName = process.env.bus_name
const topicArn = process.env.restaurant_notification_topic

module.exports.handler = async (event) => {
  const order = event.detail
  const snsReq = {
    Message: JSON.stringify(order),
    TopicArn: topicArn
  };
  await sns.publish(snsReq).promise()

  const { restaurantName, orderId } = order
  console.log(`notified restaurant [${restaurantName}] of order [${orderId}]`)

  await eventBridge.putEvents({
    Entries: [{
      Source: 'big-mouth',
      DetailType: 'restaurant_notified',
      Detail: JSON.stringify(order),
      EventBusName: busName
    }]
  }).promise()

  console.log(`published 'restaurant_notified' event to EventBridge`)
}
```

This `notify-restaurant` function would be trigger by `EventBridge`, and by the `place_order` event that we publish from the `place-order` function.

Remember that in the `place-order` function we published `Detail` as a JSON string:

```javascript
await eventBridge.putEvents({
  Entries: [{
    Source: 'big-mouth',
    DetailType: 'order_placed',
    Detail: JSON.stringify({
      orderId,
      restaurantName,
    }),
    EventBusName: busName
  }]
}).promise()
```

However, when `EventBridge` invokes our function, `event.detail` is going to be an object, and it's called `detail` not `Detail` (one of many inconsistencies that you just have to live with in AWS...)

Our function here would publish a message to the `RestaurantNotificationTopic` SNS topic to notify the restaurant of a new order. And then it will publish a `restaurant_notified` event.

But we still need to configure this function in the `serverless.yml`.

6. Modify `serverless.yml` to add a new `notify-restaurant` function

```yml
notify-restaurant:
  handler: functions/notify-restaurant.handler
  events:
    - eventBridge:
        eventBus: arn:aws:events:#{AWS::Region}:#{AWS::AccountId}:event-bus/order_events_${self:provider.stage}
        pattern:
          source:
            - big-mouth
          detail-type:
            - order_placed
  environment:
    bus_name: !Ref EventBus
    restaurant_notification_topic: !Ref RestaurantNotificationTopic
```

In case you're wondering why we aren't using `!GetAtt EventBus.Arn` in the event source definition, it's because the Serverless framework only accepts a string here.

If you have read the Serverless framework [docs](https://serverless.com/framework/docs/providers/aws/events/event-bridge#using-a-different-event-bus) on EventBridge, then you might also be wondering why I didn't just let the Serverless framework create the bus for us.

That is a very good question!

The reason is that you generally wouldn't have a separate event bus per microservice. The power of `EventBridge` is that it gives you very fine-grained filtering capabilities and you can subscribe to events based on its content such as the type of the event (usually in the `detail-type` attribute).

Therefore you typically would have a centralized event bus for the whole organization, and different services would be publishing and subscribing to the same event bus. This event bus would be provisioned by other projects that manage these shared resources (as discussed before). Which is why it's far more likely that your `EventBridge` functions would need to subscribe to an existing event bus by ARN.

As for the subscription pattern itself, well, in this case we're listening for only the `order_placed` events published by the `place-order` function.

To learn more about content-based filtering with EventBridge, have a read of [this post](https://www.tbray.org/ongoing/When/201x/2019/12/18/Content-based-filtering) by Tim Bray.

7. Modify `serverless.yml` to add the permission to `sns:Publish` to the SNS topic, under `provider.iamRoleStatements`

```yml
- Effect: Allow
  Action: sns:Publish
  Resource: !Ref RestaurantNotificationTopic
```

</p></details>

<details>
<summary><b>Add integration test for notify-restaurant function</b></summary><p>

1. Modify `steps/when.js` to add a `we_invoke_notify_restaurant` function

```javascript
const we_invoke_notify_restaurant = async (event) => {
  if (mode === 'handler') {
    await viaHandler(event, 'notify-restaurant')
  } else {
    throw new Error('not supported')
  }
}
```

and again, don't forget to add it to the list of exported methods

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order,
  we_invoke_notify_restaurant
}
```

3. Add a file `notify-restaurant.tests.js` to the `test_cases` folder

4. Modify `test_cases/notify-restaurant.tests.js` to the following

```javascript
const { init } = require('../steps/init')
const when = require('../steps/when')
const AWS = require('aws-sdk')
const chance = require('chance').Chance()
console.log = jest.fn()

const mockPutEvents = jest.fn()
AWS.EventBridge.prototype.putEvents = mockPutEvents
const mockPublish = jest.fn()
AWS.SNS.prototype.publish = mockPublish

describe(`When we invoke the notify-restaurant function`, () => {
  if (process.env.TEST_MODE === 'handler') {
    beforeAll(async () => {
      await init()

      mockPutEvents.mockClear()
      mockPublish.mockClear()

      mockPutEvents.mockReturnValue({
        promise: async () => {}
      })
      mockPublish.mockReturnValue({
        promise: async () => {}
      })

      const event = {
        source: 'big-mouth',
        'detail-type': 'order_placed',
        detail: {
          orderId: chance.guid(),
          userEmail: chance.email(),
          restaurantName: 'Fangtasia'
        }
      }
      await when.we_invoke_notify_restaurant(event)
    })

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
    it('no acceptance test', () => {})
  }
})
```

Notice that all the test cases are wrapped inside a big `if` condition. It's weird, I know.. Ignore it for now, we'll talk about it shortly.

5. Run integration tests

`npm run test`

and see that the new test is failing

```
 FAIL  tests/test_cases/notify-restaurant.tests.js

  ● Console

    console.log tests/steps/init.js:26
      AWS credential loaded
    console.log functions/notify-restaurant.js:19
      notified restaurant [Fangtasia] of order [cccf6190-9768-51ac-9435-c4ca101a6018]
    console.log functions/notify-restaurant.js:30
      published 'restaurant_notified' event to EventBridge


  ● When we invoke the notify-restaurant function › Should publish message to SNS

    TypeError: Cannot read property 'body' of undefined

      13 |   const response = await handler(event, context)
      14 |   const contentType = _.get(response, 'headers.content-type', 'application/json');
    > 15 |   if (response.body && contentType === 'application/json') {
         |                ^
      16 |     response.body = JSON.parse(response.body);
      17 |   }
      18 |   return response

      at viaHandler (tests/steps/when.js:15:16)

  ● When we invoke the notify-restaurant function › Should publish event to EventBridge

    TypeError: Cannot read property 'body' of undefined

      13 |   const response = await handler(event, context)
      14 |   const contentType = _.get(response, 'headers.content-type', 'application/json');
    > 15 |   if (response.body && contentType === 'application/json') {
         |                ^
      16 |     response.body = JSON.parse(response.body);
      17 |   }
      18 |   return response

      at viaHandler (tests/steps/when.js:15:16)
```

This is because our `notify-restaurant` function doesn't return any response because it doesn't need to. But the `when.viaHandler` function kinda expects a response object with `body`.

6. Modify `steps/when.js` to update the `viaHandler` function to handle this

```javascript
const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (_.get(response, 'body') && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

7. Rerun integration tests

`npm run test`

and see that all tests are passing now

```
 PASS  tests/test_cases/notify-restaurant.tests.js
 PASS  tests/test_cases/get-index.tests.js
 PASS  tests/test_cases/get-restaurants.tests.js
 PASS  tests/test_cases/place-order.tests.js
 PASS  tests/test_cases/search-restaurants.tests.js (6.221s)
  ● Console

    console.info functions/search-restaurants.js:24
      this is a new secret


Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        6.694s
```

</p></details>

<details>
<summary><b>Acceptance test for notify-restaurant function</b></summary><p>

We can publish an `order_placed` event to the EventBridge event via the AWS SDK to execute the deployed `notify-restaurant` function. Since this function publishes to both SNS and EventBridge, we have the same conumdrum in verifying that it's producing the expected side-effects as the `place-order` function.

For now, we'll take a short-cut and skip the test altogether. Notice that the test cases are all wrapped inside an `if` statement already

```javascript
if (process.env.TEST_MODE === 'handler') {
  ...
} else {
  it('no acceptance test', () => {})
}
```

so they're only executed when you run the integration tests.

The `no acceptance test` is a dummy test, it's only there because Jest errors if it doesn't find a test in a module. So without it, the acceptance tests would fail because Jest the `search-restaurants.tests.js` module doesn't contain a test.

In the next couple of modules, we'll come back and address this properly.

</p></details>

<details>
<summary><b>Peeking into SNS and EventBridge messages</b></summary><p>

While working on these changes, we don't have a way to check what our functions are writing to SNS or EventBridge. This is a common problem for teams that leverage these services heavily. To address this, check out the [lumigo-cli](https://www.npmjs.com/package/lumigo-cli). It has commands to [tail-sns](https://www.npmjs.com/package/lumigo-cli#lumigo-cli-tail-sns) and [tail-eventbridge-bus](https://www.npmjs.com/package/lumigo-cli#lumigo-cli-tail-eventbridge-bus) which lets you see what events are published to these services in real time.

![](/images/mod15-002.png)

![](/images/mod15-003.png)

1. Deploy the project.

`npx sls deploy`

2. Use the `lumigo-cli` to peek at both the SNS topic and the EventBridge bus.

3. Load the index page in the browser and place a few orders. You should see those events show up in the `lumigo-cli` terminals.

</p></details>
