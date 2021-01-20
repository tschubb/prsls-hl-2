# Module 2: Serverless Framework 101

## Installation

Follow the instruction and install the CLI from [http://serverless.com/](http://serverless.com/).

## Create a Serverless project

**Goal:** Create and deploy AWS Lambda handler code using `serverless create` command.

<details>
<summary><b>HOW TO create serverless project</b></summary><p>

1. Create a directory for your serverless project.

    ```
    mkdir hello-world
    cd hello-world
    ```

2. Initialise the project:
    
    `npm init`
    
    Name the project accordingly and you can accept the rest of the defaults.

3. Install the `Serverless` framework as dev dependency.

    `npm install --save-dev serverless`

    Now you can run serverless using `npx sls [<args>...]`

    > _Pro tip:_ Most examples gives steps to install and run Serverless Framework globally (allowing you to directly call `serverless` in your terminal). However, global package dependency will likely to cause issues in the future between two projects depending on incompatible major versions, especially when used by build and deploy steps on your CI.

4. Create nodejs Serverless project using one of the default templates:

    `npx sls create --template aws-nodejs`

    See more information about `serverless create` command on [CLI documentation](https://serverless.com/framework/docs/providers/aws/cli-reference/create/) page.
</p></details>

<details>
<summary><b>HOW TO configure the serverless application</b></summary><p>

1. Modify the `serverless.yml` file, rename `service` to `hello-world-` followed by your name - e.g. `hello-world-yancui`.

2. Go to `handler.js`, and modify the handler function return to something different, e.g.:

```javascript
module.exports.hello = async event => {
    return {
        statusCode: 200,
        body: JSON.stringify({
            message: 'hello world'
        })
    };
}
```

3. Modify the `serverless.yml` file, under `functions`, so that the definition for the `hello` function looks like this:

```yml
hello:
    handler: handler.hello
    events:
        - http:
            path: /
            method: get
```

This maps an API Gateway endpoint as the event source for our Lambda function.

</p></details>

<details>
<summary><b>HOW TO invoke function locally</b></summary><p>

1. Run `invoke local` command:

    `npx sls invoke local --function hello`

    See more information about `invoke local` command on [CLI documentation](https://serverless.com/framework/docs/providers/aws/cli-reference/invoke-local/) page.

2. Verify that the function returns the following output:

```json
{
    "statusCode": 200,
    "body": "{\"message\":\"hello world\"}"
}
```
</p></details>

<details>
<summary><b>HOW TO deploy a serverless project</b></summary><p>

1. Run `deploy` command:

    `npx sls deploy`

    See more information about `deploy` command on [CLI documentation](https://serverless.com/framework/docs/providers/aws/cli-reference/deploy/) page.

2. This creates an API in Amazon API Gateway. In the output you should see something like this:

```
endpoints:
  GET - https://xxxxx.execute-api.us-east-1.amazonaws.com/dev/
```

Curl the endpoint and see that it returns a 200 response, with the JSON payload:

```json
{
    "message": "hello world"
}
```
</p></details>

Congratulations! You have successfully successfully created and deployed your first Serverless API project.

## Exercises

<details>
<summary><b>Add another function</b></summary><p>

1. Modify the `serverless.yml` file and add another function under the `functions` section.

2. Map the function to another `GET` HTTP endpoint

3. Deploy and curl the new endpoint

</p></details>

<details>
<summary><b>Deploy to another stage/environment</b></summary><p>

1. Deploy the project to a `test` stage (aka environment) with `npx sls deploy --stage test`

2. Go to API Gateway console to see that another API has been created for the `test` stage

3. Go to the Lambda console to see the functions that been created for the `test` stage

4. Note the naming conventino the Serverless framework applies to both functions and APIs

</p></details>

<details>
<summary><b>Add description to the functions</b></summary><p>

1. Consult the [Serverless framework docs](https://serverless.com/framework/docs/providers/aws/guide/serverless.yml/) to see all the different configuration options available

2. Modify the `serverless.yml` and add descriptions to the functions

3. Deploy the functions with `npx sls deploy`

4. Go to the Lambda console to see the functions have been updated with descriptions

</p></details>

<details>
<summary><b>Customize the function names</b></summary><p>

The Serverless framework enforces a naming convention, but you can override the convention.

1. Consult the [Serverless framework docs](https://serverless.com/framework/docs/providers/aws/guide/serverless.yml/) to see how you can override function names

2. Deploy the functions with `npx sls deploy`

3. Go to the Lambda console to see the functions have been renamed

</p></details>

<details>
<summary><b>Removing the project</b></summary><p>

1. Delete the deployed functions and APIs with `npx sls remove`

2. Go to the Lambda console to see the deployed functions are deleted

3. Go to the API Gateway console to see the deployed APIs are deleted

4. Go to the IAM console to see the IAM execution roles for the functions are deleted

5. Go to the CloudFormation console to see the CloudFormation stacks are deleted

</p></details>