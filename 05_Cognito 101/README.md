# Module 5: Cognito 101

## Create a new Cognito User Pool

**Goal:** Set up a new Cognito User Pool by hand

Don't worry, we'll do this properly using Infrastrcture-as-Code (Iac) in a minute. But I'd like to give you a chance to see all the different configuration options and get a sense of what Cognito can do for you.

<details>
<summary><b>Add a new Cognito User Pool</b></summary><p>

1. Go to the Cognito console

2. Click `Create a user pool`

3. Use the name `workshop-` followed by your name, e.g. `workshop-yancui`

![](/images/mod05-001.png)

4. Click `Step through settings`

5. Tick `Also allow sign in with verified email address`, `family name` and `given name`

![](/images/mod05-002.png)

6. Click `Next step`

7. Accept all the default settings in the next screen

![](/images/mod05-003.png)

8. Click `Next step`

9. Accept all the default settings in the next screen

![](/images/mod05-004.png)

10. Click `Next step`

11. Accept all the default settings in the next screen

![](/images/mod05-005.png)

12. Click `Next step`

13. Ignore tags, and click `Next step`

14. Choose `No` to remember user devices

![](/images/mod05-006.png)

15. Click `Next step`

</p></details>

<details>
<summary><b>Add client for web</b></summary><p>

![](/images/mod05-007.png)

1. Click `Add an app client`

2. Name the app client `web`

3. Untick `Generate client secret`.

![](/images/mod05-008.png)

4. Remove the permission for Writable Attributes for `Address` and `Profile`

![](/images/mod05-009.png)

5. Click `Create app client`

</p></details>

<details>
<summary><b>Add client for server</b></summary><p>

1. Click `Add an app client`

2. Name the app client `server`

3. Untick `Generate client secret`.

4. Tick `Enable username password auth for admin APIs for authentication (ALLOW_ADMIN_USER_PASSWORD_AUTH)`

5. Leave all the permissions as is

6. Click `Create app client`

</p></details>

<details>
<summary><b>Complete the creation process</b></summary><p>

1. Click `Next step`

2. Leave all the workflow customizations

3. Click `Next step`

4. Review the settings

![](/images/mod05-011.png)

5. Click `Create pool`

6. Note the `Pool Id`, and the app client IDs for both `web` and `server`

![](/images/mod05-012.png)

![](/images/mod05-013.png)

</p></details>

**Goal:** Set up a new Cognito User Pool with CloudFormation

Ok, now that you've seen how to do it by hand, let's translate what we've done to CloudFormation.

<details>
<summary><b>Add a new Cognito User Pool resource</b></summary><p>

1. In the `serverless.yml`, under `resources` and `Resources`, add another CloudFormation resource after the `RestaurantTable`.

```yml
CognitoUserPool:
  Type: AWS::Cognito::UserPool
  Properties:
    AliasAttributes:
      - email
    UsernameConfiguration:
      CaseSensitive: false
    AutoVerifiedAttributes:
      - email
    Policies:
      PasswordPolicy:
        MinimumLength: 8
        RequireLowercase: true
        RequireNumbers: true
        RequireUppercase: true
        RequireSymbols: true
    Schema:
      - AttributeDataType: String
        Mutable: true
        Name: given_name
        Required: true
        StringAttributeConstraints:
          MinLength: "1"
      - AttributeDataType: String
        Mutable: true
        Name: family_name
        Required: true
        StringAttributeConstraints:
          MinLength: "1"
      - AttributeDataType: String
        Mutable: true
        Name: email
        Required: true
        StringAttributeConstraints:
          MinLength: "1"
```

**IMPORTANT**: this should be aligned with the `RestaurantTable`, i.e.

```yml
resources:
  Resources:
    RestaurantsTable:
      ...

    CognitoUserPool:
      ...
```

</p></details>

<details>
<summary><b>Add the web and server clients</b></summary><p>

1. In the `serverless.yml`, under `resources` and `Resources`, add another CloudFormation resource after the `CognitoUserPool`.

```yml
WebCognitoUserPoolClient:
  Type: AWS::Cognito::UserPoolClient
  Properties:
    ClientName: web
    UserPoolId: !Ref CognitoUserPool
    ExplicitAuthFlows:
      - ALLOW_USER_SRP_AUTH
      - ALLOW_REFRESH_TOKEN_AUTH
    PreventUserExistenceErrors: ENABLED
```

**IMPORTANT**: this should be aligned with the `RestaurantTable` and `CognitoUserPool`, i.e.

```yml
resources:
  Resources:
    RestaurantsTable:
      ...

    CognitoUserPool:
      ...

    WebCognitoUserPoolClient:
      ...
```

2. Add the server client after the `WebCognitoUserPoolClient`:

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

Again, might the YML indentation:

```yml
resources:
  Resources:
    RestaurantsTable:
      ...

    CognitoUserPool:
      ...

    WebCognitoUserPoolClient:
      ...

    ServerCognitoUserPoolClient:
      ...
```

</p></details>

<details>
<summary><b>Add Cognito resources to stack Output</b></summary><p>

We have added a couple of CloudFormation resources to our stack, let's add the relevant information to our stack output.

* Cognito User Pool ID
* Cognito User Pool ARN
* Web client ID
* Server client ID

1. In the `serverless.yml`, add the following to the `Outputs` section (after `RestaurantsTableName`)

```yml
CognitoUserPoolId:
  Value: !Ref CognitoUserPool

CognitoUserPoolArn:
  Value: !GetAtt CognitoUserPool.Arn

CognitoUserPoolWebClientId:
  Value: !Ref WebCognitoUserPoolClient

CognitoUserPoolServerClientId:
  Value: !Ref ServerCognitoUserPoolClient
```

Afterwards, your `Outputs` section should look this like:

```yml
Outputs:
  RestaurantsTableName:
    Value: !Ref RestaurantsTable

  CognitoUserPoolId:
    Value: !Ref CognitoUserPool

  CognitoUserPoolArn:
    Value: !GetAtt CognitoUserPool.Arn

  CognitoUserPoolWebClientId:
    Value: !Ref WebCognitoUserPoolClient

  CognitoUserPoolServerClientId:
    Value: !Ref ServerCognitoUserPoolClient
```

</p></details>

<details>
<summary><b>Deploy the project</b></summary><p>

1. Run `npx sls deploy` to deploy the new resources. After the deployment finishes, you should see the new Cognito User Pool that you added via CloudFormation.

![](/images/mod05-014.png)

2. Now that we don't need it anymore, delete the Cognito User Pool that you created by hand. Feel free to compare the two side-by-side before you do, functionally they are equivalent for the purpose of this demo.

</p></details>
