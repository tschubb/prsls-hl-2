# Module 26: Powertuning functions for best performance and cost ratio

So far, we have stuck with the default memory settings (1024MB) for all of our functions.

And since the amount of memory you allocate to the function proportionally affects its CPU power, network bandwidth, and cost per 100ms invocations (see [here](https://aws.amazon.com/lambda/pricing/) for more details on Lambda pricing). One of the simplest cost optimization you can do on Lambda reducing its memory allocation!

If all your function is doing is making a request to DynamoDB and wait for its response then more CPU is not gonna improve its performance since its CPUs are just sitting idle.

If I was to guess, I'd say all of our functions can run comfortably on a much lower memory setting than the default 1024MB. But, luckily for you, I don't rely on guesses, I use data to drive technical decisions!

Alex Casalboni's [aws-lambda-power-tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) lets us collect performance information to find the best memory allocation for our workload. However, to use it, you need to deploy a Step Functions state machine, and invoke it yourself. Which is kinda clumsy.

Which is why I wrapped it into a single command you can run with the [lumigo-cli](https://www.npmjs.com/package/lumigo-cli#lumigo-cli-powertune-lambda), which also manages the deployment and upgrading of Alex's state machine too.

<details>
<summary><b>Manually tune functions</b></summary><p>

1. Install the `lumigo-cli` (if you haven't already) by running `npm i -g lumigo-cli`

2. To find the optimal memory setting for `get-restaurants` function, we need to generate a payload to invoke the function with. Fortunately, the function doesn't actually use anything from the invocation event, so `{}` would do.

Run `lumigo-cli powertune-lambda -h` to see what configuration options you have.

Now, figure out what's the full name of your `get-restaurants` function, it should be something like `workshop-xxx-dev-get-restaurants` where `xxx` is the value of the `custom.name` field in the `serverless.yml`.

For example, mine is `workshop-yancui-dev-get-restaurants`.

Now, run the following (replace `xxx`)

```
lumigo-cli powertune-lambda -r us-east-1 -n workshop-xxx-dev-get-restaurants --payload '{}' --strategy balanced
```

**NOTE**: there are 3 strategies - `cost`, `speed` and `balanced`, usually you should go with `balanced` as `cost` would always give you 128MB and `speed` would generally favor higher memory allocations even when there are marginal gains.

You should see something like this:

```
checking the aws-lambda-power-tuning SAR in [us-east-1]
the latest version of aws-lambda-power-tuning SAR is 3.2.5
looking for deployed CloudFormation stack [serverlessrepo-lumigo-cli-powertuning-lambda] in [us-east-1]
stack is deployed but is running an outdated version [3.2.4]
CloudFormation template has been generated
waiting for SAR deployment to finish...
.....
SAR deployment completed
the State Machine is arn:aws:states:us-east-1:374852340823:stateMachine:powerTuningStateMachine-7rCSp1kyv4TF
State Machine execution started
execution ARN is arn:aws:states:us-east-1:374852340823:execution:powerTuningStateMachine-7rCSp1kyv4TF:ede4181f-21f0-47f1-8a59-cd750c637777
..
{
  "power": 512,
  "cost": 8.332e-7,
  "duration": 23.76583333333334,
  "stateMachine": {
    "executionCost": 0.0003,
    "lambdaCost": 0.0010964911999999999,
    "visualization": "https://lambda-power-tuning.show/#gAAAAQACAAQABsAL;IQnhQnkpUUJtIL5B7+6XQWNJiEHfFoRB;EanfNBGp3zQRqV81EanfNc2+JzYpQKQ2"
  },
  "functionName": "workshop-yancui-dev-get-restaurants"
}

? Do you want to open the visualization to see more results? (Use arrow keys)
❯ yes 
  no 
```

If you enter yes, then you should see something like this:

![](/images/mod26-001.png)

As you can see, `512MB` gives us the best bang for our buck, and there are negligible gains by going up to `1024MB`. Let's take the 50% cost saving and run!

3. Now that we know the best memory setting for the `get-restaurants` function, let's go back to the `serverless.yml` and update its configuration.

e.g.

```yml
get-restaurants:
  handler: functions/get-restaurants.handler
  memorySize: 512
  ...
```

4. Repeat this for the `get-index` function.

```
lumigo-cli powertune-lambda -r us-east-1 -n workshop-xxx-dev-get-index --payload '{}' --strategy balanced
```

![](/images/mod26-002.png)

![](/images/mod26-003.png)

5. Now, see if you can figure out what payload you need to invoke `search-restaurants`, `place-order` and `notify-restaurant` with, so you can tune these functions too ;-)

</p></details>

<details>
<summary><b>Automatically tune functions during CI</b></summary><p>

So far, we've explored how to use the [lumigo-cli](https://www.npmjs.com/package/lumigo-cli) to figure out what's the best memory setting for our function.

But that's manual and laborious, and what if we change something in our code and forget to rerun this process? We could accidentally leave our function underpowered or end up overpaying for them.

If you run `lumigo-cli powertune-lambda -h` you will notice that there's an `autoOptimize` option! This instructs the state machine to automatically apply the best memory setting to the function at the end of the process.

![](/images/mod26-004.png)

So, we can actually do this as part of our CI/CD pipeline - deploy the functions, then run `lumigo-cli powertune-lambda --autoOptimize ...` to automatically tune the function so it's never mis-configured!

HOWEVER, this might be a risky thing to do in production! Especially if you have functions that produces side-effects (e.g. the `place-order` function) like writing data to DynamoDB, etc. Instead, what you could do, is to script it as part of your CI to:

1. run `lumigo-cli powertune-lambda` and send the response to an output file with the `--outputFile` flag, and suppress the visualization option with the `--noVisualization` flag

2. parse the output JSON file

3. if the configuration for a file is different to what's in the `serverless.yml`, then update the configuration in the `serverless.yml`

4. commit the change, and auto-create a PR

Check out [this post](https://theburningmonk.com/2020/03/how-to-optimize-lambda-memory-size-during-ci-cd-pipeline/) for an example on how to extract the best memory setting from the output.

</p></details>
