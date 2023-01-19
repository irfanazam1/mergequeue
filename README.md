# Merge Queue Demo

## Merge Queue

Merge queues prevent merging broken pull requests by serializing or ordering their merges. Merging broken pull requests can happen when outdated pull requests are being merged in their base branch. 

Essentially a merge queue do the following things,
 - Orders or serialize the merges using a queue.
 - Update the PR with the base branch and runs CI checks on it before merging it to the base branch. It make sure only a valid commit is merged into      the base branch.
 - Removes broken PRs from the queue. Broken PRs means the commits failing CI checks or having merge conflicts with the base branch.

Benefit:
Keeps the main/base branch green.

## Why Queues?

### Situation:
- The base branch (e.g., main) has its CI testing passing correctly.
- A PR is created, which also passes the CI.

The state of the repository can be represented like this:
![image](https://user-images.githubusercontent.com/59575775/213455560-a0a5d37d-deff-48af-9f2f-d9d42b342a1b.png)

While the pull request is open, another commit is pushed to main — let's call it new commit. That new commit can be pushed directly to main or merged from another pull request; it doesn't matter.

The tests are run against the main branch by the CI, and they pass. The state of the repository and its continuous integration system can be now described like this:
![image](https://user-images.githubusercontent.com/59575775/213455909-442b4af7-8cc8-455c-b3a6-8a9689b142bd.png)

Base branch adds a new commit. The pull request is still marked as valid by the continuous integration system since it did not change. As there is no code conflict, the pull request is considered as mergeable by GitHub: the merge button is green.

If you click that merge button, this is what might happen:
![image](https://user-images.githubusercontent.com/59575775/213456170-5f2271c0-dd43-43ba-aca6-4d9f0caf0d92.png)

Base branch is broken. As a new merge commit is created to merge the pull request, it is possible that the continuous integration testing fails. Indeed, the continuous integration did not test the pull request with the new commit added to the base branch. This new commit might have introduced some new tests in the base branch while the pull request was open. That pull request may not have the correct code to pass this new test.


## Using Queues

Using a merge queue solves that issue by updating any pull request that is not up-to-date with its base branch before being merged. That forces the continuous integration system to retest the pull request with the new code from its base branch.

When multiple pull requests are mergeable, they are scheduled to be merged sequentially, and are updated on top of each other. The pull request branch update is only done when the pull request is ready to be merged by the engine, e.g., when all the merge_conditions are validated.

That means that when a first pull request has been merged, and the second one is outdated like this:

![image](https://user-images.githubusercontent.com/59575775/213458215-9fb5367c-94ea-473c-842a-61fb3fdbf1f8.png)


Merge queue will make sure the pull request #2 is updated with the latest tip of the base branch before merging:

![image](https://user-images.githubusercontent.com/59575775/213458359-6107ffcc-71c4-4a6a-9cf1-28da35641b14.png)


## Strict Merge
That way, there's no way to merge a broken pull request into the base branch when using merge queues.

## Github Merge Queue Implementations
- Bors tech merge queue bot (https://bors.tech/)
- Github merge queue (https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)
- Mergify (https://docs.mergify.com/actions/queue/)

## Demo
The demo uses a mergify queue. Mergify is a paid github bot/app for enterprise but free for invidual/personal use.
- Installation: https://docs.mergify.com/getting-started/#installation
  - You create a personal account on www.mergify.com
  - Link your account with mergify by accepting the github app consent while adding mergify as an app.
  - You either give permissions to all the repos or select specific ones.
  - Mergigy requires permissions to create temporary branches, create commits, push to branches, delete branches etc.

![image](https://user-images.githubusercontent.com/59575775/213461070-c1ac3d3a-625c-4feb-b8c6-32aaa104e32b.png)

- Configuration: You provide a configuration file with a name .mergigy.yml, .mergify/config.yml, or .github/mergify.yml

# The Rules

The configuration file is composed of a main key named pull_request_rules which contains a list of rules.
Each rule is composed of 3 elements:
- A name that describes what the rule does. It's not interpreted by Mergify and can be anything you like that helps you identify the rule.
- A list of conditions. Each conditions must match for the rule to be applied.
- A list of actions. Each action will be applied as soon as the pull request matches the conditions.

# Queue Rules
The rules used for this demo
```
queue_rules:
   - name: default
     conditions:
       - check-success='SDK Tests'
       - check-success='Benchmarks'
       - check-success='SmokeTests'
       - check-success='Docker Image'

 pull_request_rules:
   - name: merge using the merge queue
     conditions:
       - base=main
       - approved-reviews-by>=1
     actions:
       queue:
         name: default
```
