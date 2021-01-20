# Module 11: CI/CD with Circle CI

**Goal:** Configure CI with Circle CI

<details>
<summary><b>Add CircleCi config</b></summary><p>

1. Add a folder called `.circleci` at the project root.

2. Add a file called `config.yml` in this new `.circleci` folder and paste the following into it.

```yml
version: 2.1
orbs:
  node: circleci/node@1.1

jobs:
  integration-test:
    docker:
      - image: circleci/node
    steps:
      - checkout
      - node/with-cache:
          steps:
            - run: npm ci
            - run: npm run test # run our integration tests
  deploy:
    docker:
      - image: circleci/node
    steps:
      - checkout
      - node/with-cache:
          steps:
            - run: npx sls deploy
  acceptance-test:
    docker:
      - image: circleci/node
    steps:
      - checkout
      - node/with-cache:
          steps:
            - run: npm run acceptance

workflows:
   version: 2
   dev:
     jobs:
       - integration-test
       - deploy:
          requires:
            - integration-test
       - acceptance-test:
          requires:
            - deploy
```

Commit and push the change to your repo and watch the build kick off.

And if everything goes well, it should complete successfully.

![](/images/mod11-001.png)

</p></details>
