---
title: Responding to Heroku's shutdown of their GitHub Integration
description: A list of problems and solutions I ran into and came up with to keep a project running after Heroku shut down their GitHub integration in 2022 due to a security incident.
---

```
author: @ekilah
created_at: 2022-04-20
updated_at: 2022-04-21
```

# [Responding to Heroku's shutdown of their GitHub Integration](#)

On April 15, 2022, [GitHub announced](https://github.blog/2022-04-15-security-alert-stolen-oauth-user-tokens/) an attack that seemed to be abusing GitHub OAuth tokens issued to a few services, including Heroku. Over the next 48 hours, [Heroku decided](https://status.heroku.com/incidents/2413) to revoke all of the possibly-breached tokens and disable their GitHub integration completely. 

We don't yet know the size of this breach or how justified Heroku's move was, but for now, the effect is the same: all workflows relying on Heroku's GitHub integration are currently... pretty broken. 

Here's a short list of things that don't work during this outage, which shows just how deep the Heroku integrates with / relies on GitHub connections:

- Automatic deploys of a GitHub branch (e.g. `main` or `master`)
- Review Apps (apps Heroku can build when a PR is opened)
- Heroku CI cannot create new runs (automatically or manually) or see GitHub branch list
- Heroku Button: unable to create button apps from private repositories
- ChatOps: unable to deploy or get deploy notifications
- Any app with a GitHub integration may be affected

## [Existing solutions (lack thereof)](#existing-solutions-lack-thereof)

While Heroku figures out their own issues, end users are left to figure out how to get back to their own work mostly on their own. Heroku's status page links to these two articles but mostly leaves out any other helpful details:

- [Integrating with Version Control Providers Besides GitHub](https://devcenter.heroku.com/articles/integrating-with-version-control-providers-besides-github)
- [Deploying with Git](https://devcenter.heroku.com/categories/deploying-with-git)

The internet seems to lack many posts about how to use Heroku without a GitHub OAuth connection, and that feels fairly reasonable; the GitHub integration is extremely easy to add, is required for many features, has been around for a long time, and works quite well. Until it doesn't, that is. 😉

## [What this post covers](#what-this-post-covers)

In this post, I will go through the steps I took to get a Heroku build pipeline back up and running this week. The end result isn't perfect, and I will still look to re-enable the GitHub integration on Heroku's dashboard when we can, ultimately reverting a lot of these changes as a result. But, in the meantime, these changes have gotten us past the blockers Heroku's outage presented.

The pieces we used that I fixed: 

* Automatic deployment of a GitHub branch
    * [`master` for production and prod-mirroring environments](#automatic-deploys-to-production)
    * [several other environments like `staging`](#automatic-deploys-to-other-environments)
* [Creating Review Apps when PRs are opened](#creating-review-apps-when-prs-are-opened)
* [Updating Review Apps when PRs are modified](#updating-review-apps-when-prs-are-modified)

The workflow I was fixing up used both GitHub Actions and CircleCI, which meant I had some choice regarding which tool I could use to solve these problems (_something_ needs to be triggered by changes to branches, after all, and using tools we already integrate with makes this easier/cheaper).

I chose CircleCI, mostly because I had been working on our CircleCI configuration recently for other reasons, though I think the ideas here could easily be used in GHA or other CI tools.

## [Automatic deploys to production](#automatic-deploys-to-production)

First things first: unbreak prod. We've got an application running on Heroku that people will want to push updates to on Monday morning, and chaos will ensue if `master` is not deployed as they expect it to be!

Of course, this issue was first reported on a Friday afternoon, which meant I was checking Slack and Heroku's status page all weekend to see what might come of this. And on Sunday morning, I saw that Heroku finally decided to disable their GitHub integration completely, for all customers. 

I had anticipated this result as a possible outcome, but it wasn't clear how exactly the breakage would affect us until they actually went through with it. 

So, I went to check our production environment and compare the current, live release with `HEAD` of our `master` branch, and voila, there was already a mismatch (someone merged some code on Saturday night that never deployed). 

With the problem identified, the next steps were to dive into Heroku's documentation to figure out what it takes to trigger a deploy manually, and then to figure out how to do so via CircleCI.

### [Triggering a deploy manually](#triggering-a-deploy-manually)

I found these two docs to be most helpful in this process, so I'll link to them directly:

- [CircleCI Deployment docs: A Simple Example Using Heroku](https://circleci.com/docs/2.0/deployment-integrations/#a-simple-example-using-heroku)
- [The Heroku Orb's docs on CircleCI](https://circleci.com/developer/orbs/orb/circleci/heroku)

Orbs, if you don't know, are bits of CircleCI config bundled up for reuse. Including one in your config file is easy, and doing so gives you access to jobs/commands/executors/etc that the Orb defines.

Luckily for us, Heroku maintains an Orb with decent-enough documentation for use with CircleCI, so deploying to Heroku from a CircleCI job isn't all that hard.

Include this at the top level of your `.circleci/config.yml` file to import the Orb:

```yml
orbs:
  heroku: circleci/heroku@1.2.6
```

Then, define a new `job` in your `workflow`. The workflow in this example is named `build`, but yours can be named whatever you like. Our new job, `deploy_master_to_production`, looks like this:

```yml
workflows:
  build:
    jobs:
      - deploy_master_to_production:
          filters:
            branches:
              only: master
          requires:
            - create_caches
            - production_code_checks
          context: production
```

I used the branch filtering feature here to make sure this job will only run when CI is triggered for the `master` branch, since that is the branch we use for production.

> Other unimportant tidbits about this config, if you are interested:
> 
> - `context: production` tells CircleCI to load a set of environment variables grouped into something they call a "Context," which is named and configured on their dashboard. 
> - `requires:` defines other jobs this job relies on / jobs that must run before this one. The contents of those jobs aren't relevant here. For completeness, they cache things like `node_modules` for use by other jobs, and run some checks like type checks, respectively. You'll see these jobs running first in some screenshots later.

The above config tells CircleCI about a new job, `deploy_master_to_production`, and when & how we want it to run. Next, we have to define what this job does, in the `jobs` top-level key of the config.

```yml
jobs:
  deploy_master_to_production:
    docker:
      - image: cimg/node:16.2.0
    resource_class: small
    working_directory: ~/repo

    steps:
      - checkout
      - heroku/install
      - heroku/check-authentication
      - run:
          name: Set Heroku Env Vars for this branch
          command: |
            echo 'export HEROKU_APP_NAME="some-heroku-app-name"' >> $BASH_ENV
      - heroku/deploy-via-git
```

Let's walk through this config step-by-step:

- First we define some details about the machine type to use and the `working_directory` this job should use on the machine.
    - I used `small`, CircleCI's smallest resource class, because this job just does some simple HTTP/git commands, which don't need a lot of computing power.
- Then you define the `steps` for the job, which is a list of `commands`.
    - `checkout` is something CircleCI provides, which clones your git repo for you.
    - `heroku/install` is from the Heroku Orb, and it installs the Heroku CLI tools so you can use them.
    - `heroku/check-authentication` verifies that `HEROKU_API_KEY` is set in the environment, which is required for the Heroku CLI operations we're about to use. Nice as a sanity check.
        - I added this environment variable via CircleCI's dashboard to my entire project. You can also set it within a `Context` as described above.
    - The custom command via `run` here sets `HEROKU_APP_NAME` into another environment variable, which will be used by `heroku/deploy-via-git`
        - The way we did this looks odd, but this whole "insert an `export` command into `$BASH_ENV`" bit gets around the fact that each `step` runs in isolation, which prevents you from doing something simpler like `export HEROKU_APP_NAME=...`. ([docs link](https://circleci.com/docs/2.0/env-vars/#using-parameters-and-bash-environment))
    - `heroku/deploy-via-git` will take `HEROKU_APP_NAME` and `HEROKU_API_KEY` and deploy the current branch (reading from CircleCI's `CIRCLE_BRANCH` env var) to the given Heroku app.

    
And that's it! This configuration solves our first major problem, and we can now deploy to production again as desired.

## [Automatic deploys to other environments](#automatic-deploys-to-other-environments)

With production unblocked, the next step was to unbreak our staging environments. We have several staging environments, one named `staging` and others with this name pattern: `staging-*`.

Each of these environments have their own `git` branch of the same name, and their own frontend and backend. We want to make sure that changes to e.g. the `staging` branch for our frontend get deployed automatically to our `staging` frontend server, the same way that we just got `master` deploying to our production frontend server.

Sounds easy, right? Didn't we just do that for `master`? Sort of.

The biggest problem with applying the earlier fix for `master` to our staging envs revolves around a long-requested CircleCI feature: always triggering CI on multiple branches, but not every commit.

To curtail wasted CI cycles, CircleCI has a setting to only run CI on open PRs, which skips commits pushed that have no associated PR yet. This is essential for large teams, since CI has limited resources that would be wasted on commits not yet ready for tests/review. That setting has a built-in exception so that your default branch is always built (you usually want CI to run on `master` to make sure merges pass tests on it, and/or to deploy it).

The problem is that many projects have _multiple_ "default branches", meaning branches other than `master` that they always want CI to run on. CircleCI [does not support this on their dashboard](https://ideas.circleci.com/cloud-feature-requests/p/allow-branch-whitelist-to-override-only-build-pull-requests), though there is apparently [a way for their support team](https://discuss.circleci.com/t/unable-to-limit-builds-to-only-prs-and-multiple-whitelisted-branches/39561/2) to do this for you. ~~I haven't heard back from them on this yet, but I did ask.~~

**Update:** CircleCI got back to me within a couple of days, and they were able to do a pattern-based fix on their end, so now my default branch patterns are `master`, `staging`, and `staging-*`, which is exactly what I wanted. The quoted section below goes into my workaround before they applied this setting for me.

> Without this feature, and with the "Only build pull requests" feature enabled, our staging branches won't have CI run on them. Not great if I want CI to trigger builds on those branches. The good news is that you can manually trigger CI builds for any branch on CircleCI's dashboard. This is pretty lame compared to automatic deploys like we had before, but a small cost to pay for now.
> > In the future, we could probably automate this by calling the CircleCI API to trigger the staging CI pipeline, but that felt like a whole lot of work (presumably I need to do this via something else like GHA?) for not a lot of gain in the short term. CircleCI should really solve this.

Here's the config, for completeness. We define a new `job` at the top level of the config, along with a reusable `executor` since the machine details are the same as our other job:

```yml
executors:
  heroku_build_machine:
    docker:
      - image: cimg/base:stable
    resource_class: small
    working_directory: ~/repo
    
jobs:
  deploy_staging_to_heroku:
    executor: heroku_build_machine
    steps:
      - checkout
      - heroku/install
      - heroku/check-authentication
      - run:
          name: Set Heroku Env Vars for this branch
          command: |
            echo 'export HEROKU_APP_NAME="your-app-prefix-$CIRCLE_BRANCH"' >> $BASH_ENV
      - heroku/deploy-via-git:
          force: true
```

This is mostly the same as `deploy_master_to_production` from earlier, except the force push (we needed it here) and that `HEROKU_APP_NAME` is now dynamic.

Our Heroku app names for staging envs follow a pattern, where the branch name is at the end of the app name, so I was able to insert `CIRCLE_BRANCH` into `HEROKU_APP_NAME`. With this setup, all of our staging branches can use this same job, and no changes are required here for future staging envs:

```yml
workflows:
  build:
    jobs:
      - deploy_staging_to_heroku:
          filters:
            branches:
              only: /staging(-\w+)?/
          context: staging servers
```

## [Creating Review Apps when PRs are opened](#creating-review-apps-when-prs-are-opened)

Ok, so now that production and staging servers are both up and running again, we can move on to fixing the second half of the breakage: Review Apps.

We use the Review Apps feature quite frequently. It is very convenient: every PR has a build made for it, where your changes are visible and can be easily sent to designers or others without asking them to run the code locally. 

This feature also relies on Heroku's GitHub integration, since it needs to know when PRs are opened and modified. The GitHub integrated version would even post build status notifications to your PRs and add a handy "View deployment" button on your PR once it was ready.

I haven't gotten around to replicating these nice-to-have UX features, but I definitely wanted Review Apps to be created for new PRs again. At first, things didn't look so promising. According to [Heroku's Review Apps documentation](https://devcenter.heroku.com/articles/github-integration-review-apps): 

> Note that your app must enable both Heroku Pipelines and GitHub integration to use review apps.

Luckily for us, Heroku's own API documentation seemed to disagree. So, off to the [Review App API docs](https://devcenter.heroku.com/articles/platform-api-reference#review-app). These docs describe a few APIs that allow us to create, query for a list of, and delete Review Apps. Great!

However, I immediately noticed a problem: the Create API requires you to pass a publicly-accessible URL to a `tarball` of the code you wish to deploy to a Review App. Naturally, this won't work well with private repositories.

Googling around led me to [this article](https://help.heroku.com/5WGYZ74Q/what-should-i-use-for-the-value-of-source_blob-when-creating-apps-via-the-platform-api) on Heroku's support site, which suggested using HTTP Basic Authentication via `https://username:token@api.github.com`-formatted URLs to get around this issue. Let's try it.

```json
{
  "message": "Not Found",
  "documentation_url": "https://docs.github.com/rest/reference/repos#download-a-repository-archive"
}
```

No dice. Seems like GitHub disabled this method, though their docs [still describe Basic Authentication working with OAuth tokens](https://docs.github.com/en/rest/overview/other-authentication-methods#via-oauth-and-personal-access-tokens), so I'm not sure what the problem was here. I'm not sure if this was user error or not, looking back, but either way, it didn't work for me.

So, instead, I crafted a workaround: `cURL` the GitHub API for the `tarball`, grab the temporary, public URL in the response header's `location` key, and pass it to Heroku's API. Works like a charm, with some extra work to strip the header name and line-endings from the response:

```bash
TARBALL_LOCATION=$(curl -I -u $GITHUB_USERNAME:$GITHUB_API_TOKEN https://api.github.com/repos/$GITHUB_ORG/$GITHUB_REPO/tarball/$CIRCLE_BRANCH | grep -Fi location: | tr -d '\r' | sed -e 's/location: //gi')
```

Here's a new, reusable `command` in our CircleCI config (commands are reusable steps you can use in multiple `job`s) to create a Heroku Review App:

```yml
commands:
  deploy_review_app_to_heroku_cmd:
      steps:
        - run:
            name: Create review app on Heroku
            command: |
              PR_NUMBER=$(echo $CIRCLE_PULL_REQUEST | rev | sed -e 's/\/.*$//g' | rev)
              TARBALL_LOCATION=$(curl -I -u $GITHUB_USERNAME:$GITHUB_API_TOKEN https://api.github.com/repos/$GITHUB_ORG/$GITHUB_REPO/tarball/$CIRCLE_BRANCH | grep -Fi location: | tr -d '\r' | sed -e 's/location: //gi')
              curl -X POST https://api.heroku.com/review-apps \
                -d "{
                \"branch\": \"$CIRCLE_BRANCH\",
                \"pr_number\": $PR_NUMBER,
                \"pipeline\": \"$HEROKU_PIPELINE_ID\",
                \"source_blob\": {
                  \"url\": \"$TARBALL_LOCATION\",
                  \"version\": \"$CIRCLE_SHA1\"
                },
                \"environment\": {
                  \"SOURCE_VERSION\": \"$CIRCLE_SHA1\"
                }
              }" \
                -H "Content-Type: application/json" \
                -H "Accept: application/vnd.heroku+json; version=3"\
                -H "Authorization: Bearer $HEROKU_API_KEY"
```

> This `command` uses a lot of environment variables. Here's a list of the important new ones and what they are for, for reference:
> 
> - `GITHUB_USERNAME`: the username of the user with access to the repo and for which an API token is generated
> - `GITHUB_API_TOKEN`: the GitHub API token you need to generate to use the GitHub API. You can create one at [https://github.com/settings/tokens](https://github.com/settings/tokens)
> - `PR_NUMBER`: Heroku's API takes this as an optional parameter, it makes their dashboard look nicer presumably. I parsed this out of `CIRCLE_PULL_REQUEST` which is provided by CircleCI, but is a full URL to the PR, so I had to extract the PR number out.
> - `HEROKU_PIPELINE_ID`: The pipeline you want to create the review app within. This appears in the URL when you view the pipeline on their dashboard, like `https://dashboard.heroku.com/pipelines/HEROKU_PIPELINE_ID`. 
> - `CIRCLE_SHA1`: the SHA1 hash of the commit to build. Heroku's API wants this in the `source_blob` object, and our Webpack script also wanted it in the environment for the app when it builds on Heroku, so I passed it into a custom env var named `SOURCE_VERSION` here in the optional `environment` key of this Heroku API call.


And here's our first `job` definition to use this command:

```yml
jobs:
  deploy_review_app_to_heroku:
    executor: heroku_build_machine
    steps:
      - deploy_review_app_to_heroku_cmd
```

Done! Push this up into a new PR and you'll see your Review App start building on Heroku. Note that this job will not wait for the Heroku build to actually succeed (it just calls the API and finishes), but I didn't want that anyway. I'd rather not tie up a CircleCI executor waiting for a Heroku machine to do its job. 


![review app building on heroku](/assets/img/heroku_github/review-app-building-on-heroku.png)
_Success!_

> Note that this job runs _every time_ you commit to a branch. Luckily, it takes only a second or so, and Heroku's API just returns a message saying there is already an app for this branch when we call it multiple times, and the CI job still "passes." If these things weren't true, we'd have to avoid this with some extra complexity.

## [Updating Review Apps when PRs are modified](#updating-review-apps-when-prs-are-modified)

Ok, almost done for now! The last major feature we are missing is to update an existing Review App if/when we make changes to a PR. Heroku's GitHub-integrated Review Apps feature would do this automagically, and we rely on it, so let's see what our options are there.

Going back to the [Review App API docs](https://devcenter.heroku.com/articles/platform-api-reference#review-app), it seems like the `U` in `CRUD` is missing... but no, I have to be misreading this, there _must_ be a way to update a Review App without deleting it first, right?

Nope. Not that I could find. It's very odd, because the GitHub-integrated version definitely does this, but for whatever reason, it isn't part of Heroku's public API. 

> It's possible that you can use another more generic API like we did with `heroku/deploy_via_git` for this, but the docs never mention it, and I didn't try. Seems like this should really be possible...

I also tried calling the Create endpoint a second time, but that just returns a relatively unhelpful message:

```json
{
  "id": "conflict",
  "message": "A review app already exists for the branchname_here branch"
}
```

So, to unblock this workflow, I begrudgingly added a way to delete an existing app before recreating a new one for the same branch. This is unfortunate because, unlike the update-in-place the official Heroku integration would have done here, this will actually remove the old instance of the Review App before the new one is built. That means the links you sent to a designer will stop working during the build (and our builds take several minutes to complete). 

In order to delete the review app, we need its ID, which we don't have (it is autogenerated). The Heroku API has an endpoint we can call for this, which returns a list of Review Apps and data about them, for a given pipeline. I used Ruby here to parse the JSON response and find the Review App for the current branch.

Here's the `command` definition for deleting an existing Heroku app:

```yml
commands:
  delete_review_app_on_heroku_cmd:
    steps:
      - run:
          name: Delete review app on Heroku
          command: |
            REVIEW_APP_ID=$(curl https://api.heroku.com/pipelines/$HEROKU_PIPELINE_ID/review-apps \
              -H "Authorization: Bearer $HEROKU_API_KEY" \
              -H "Content-Type: application/json" \
              -H "Accept: application/vnd.heroku+json; version=3" | \
              ruby -rjson -e 'x = JSON.parse(STDIN.read); reviewApp = x.find {|el| el["branch"] == ENV["CIRCLE_BRANCH"]}; res = reviewApp && reviewApp["id"] || "none"; puts res')
            echo $REVIEW_APP_ID
            if [ "$REVIEW_APP_ID" != "none" ]; then curl -X DELETE https://api.heroku.com/review-apps/$REVIEW_APP_ID \
              -H "Content-Type: application/json" \
              -H "Accept: application/vnd.heroku+json; version=3"\
              -H "Authorization: Bearer $HEROKU_API_KEY" ; fi
```

The last step here is to not _always_ delete the previous Heroku Review App for a given PR. I want to give the developer a choice in the matter, so they can choose if/when they want to delete the old Review App and create a new one. I chose this method to reduce the pain of pushing commits, but it will increase the work required to update a Review App a little. 

Just like our staging environments required pre-secret-CircleCi-setting-fix above, we'll require devs to go to the CircleCI dashboard, but this time, they will be choosing to approve an optional job on the existing CI pipeline for their branch.

This can be done with a feature called an "approval job" on CircleCI. Adding `type: approval` to a workflow's job definition will create a job that requires a human to click a button before other, dependent jobs run.

So, here's what that config looks like:

```yml
workflows:
  build:
    jobs:
      - approve_to_redeploy_heroku_review_app:
          type: approval
          filters:
            branches:
              ignore: /^(master)|(staging(-\w+)?)$/
          
      - redeploy_review_app_to_heroku:
          filters:
            branches:
              ignore: /^(master)|(staging(-\w+)?)$/
          requires:
            - approve_to_redeploy_heroku_review_app

jobs:
  redeploy_review_app_to_heroku:
    executor: heroku_build_machine_ruby
    steps:
      - delete_review_app_on_heroku_cmd
      - deploy_review_app_to_heroku_cmd
```

If `approve_to_redeploy_heroku_review_app` is manually approved on the CircleCI dashboard, `redeploy_review_app_to_heroku` will run, which runs our `delete` and `deploy` commands from earlier, one after the other.

![clicking approve in circleci](/assets/img/heroku_github/approval-job-in-circleci.gif)
_Manual approval. At least it is only two clicks..._

## [Conclusion](#conclusion)

The developer UX here is a whole lot worse than Heroku's native GitHub integration would produce, I admit. With more work, it might not have to be, with some automatic posts/status updates to a PR, or maybe even a button on a PR people can press. But, the important bits are present, and these changes will allow us to keep working through Heroku's security incident. I will hopefully be rolling back a lot of these changes once the incident is resolved and the GitHub integration is restored, but who knows when that will be! 

It was useful learning more about Heroku and CircleCI through the process 🙃
