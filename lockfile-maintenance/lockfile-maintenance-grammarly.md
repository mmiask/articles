# Dependency bump a day keeps the code crashes away (_a.k.a the importance of updating a lockfile_)

Coding is fun, there's no doubt about it. 

But do you know what else is? Testing! Code maintenance! Keeping dependencies in sync!

No? Only for me? Well, it might be not fun for many of you, but to keep your library/app working properly you'll need to adapt and at least try some of it.

This article focuses on a very specific task: lockfile maintenance. I'll show you:

* what `yarn.lock` is
* why you might need to do lockfile maintenance
* what is the possible solution
* what mistakes we've done so you can avoid doing them

This article does not focus on dependency versioning or the overall testing approach. These are different topics (although connected to this one) and should be handled separately

> **Warning**: all of the mentioned below is based on true events. No developers were harmed during implementation and/or for the purposes of this article

## Problem

For the last few months, I've been working at **Stoplight** as a part of **11Sigma** crew on an open-source library called [Elements](https://stoplight.io/open-source/elements/). As the time of the beta release was approaching, we've started to put more and more emphasis on testing the demos/examples that use `Elements`, not only the components themselves.

An [issue](https://github.com/stoplightio/elements/issues/989) emerged that made us [think](https://github.com/stoplightio/json-schema-viewer/pull/121) if we have our dependencies under control. In short:

> - Our library (`Elements`) used dependency A (`JSV`) with non-pinned version (`^4.0.0-beta.7`)
> - `JSV 4.0.0-beta.7` used `JST` with non-pinned version (`^1.1.0`). A new version of JST (`1.1.2`) which falls into the requirement of being `^1.1.0` included the fix that would solve our problem
> - `Elements` still crashes for us. Why?

The answer is actually pretty simple - even though both of the `JSV` and `JSV` versions are not pinned and should update upon a fresh install, our local `yarn.lock` files were blocking these updates.

Fortunately, this turned out to be a non-issue on a fresh install.

Unfortunately, that meant that we weren't testing what users are using at a given moment. Thus, we started to look for a solution...

## What is a lockfile?

_If you are familiar with this already, feel free to skip_

To understand why the topic of this article is important to you, it is necessary to know what a lockfile is and how does it work. Although it can have different names depending on whether you use `npm` or `yarn`, the premise is pretty much the same. Because I'm using `yarn` though, I'll use `yarn.lock` as an example in this article.

When you run `yarn` in your project, two things can happen:

1. A lockfile will be generated (if there isn't any)
2. Packages will be installed according to contents of an existing `yarn.lock`

### Generating `yarn.lock`

Whenever you run `yarn` (which equals `yarn install`) upon a fresh install, a `yarn.lock` file is generated. It lists the versions of dependencies that are used at the time of the installation process. That means it looks into your `package.json` and depending on the versioning syntax, it will install your project dependencies, then their dependencies, then their dependencies, and so on...

> For more info about dependency versioning check [this link](https://docs.npmjs.com/cli/v7/configuring-npm/package-json#dependencies)
 
Let's say your project uses two dependencies: `chicken` and `farm`. Both of these are external packages, over which we don't have any control
```json
// package.json (your project)

  dependencies: {
    "chicken": "^1.2.0",
    "farm": "2.3.0"
  }
```

and `farm` package uses pinned version of `chicken`:
```json
// package.json (`farm` package)

  dependencies: {
    "chicken": "1.0.0",
    (...)
  }
```

This will result in your project requiring two versions of `chicken`:
* 1.0.0 for `farm`
* ^1.2.0 as defined in your project's `package.json`. This will vary upon a fresh install depending what's the latest version after `1.2.0`

Both of these versions will be present in the `yarn.lock`

### Updating the lockfile

Things are a bit easier to explain when it comes to updating the lockfile. It can happen in 3 situations - when the dependency:

- is added,
- removed,
- modified

This can happen in two ways - via `yarn` CLI e.g.

```zsh
> yarn add PACKAGE-NAME
```
or by manually modifying the `package.json` and then running `yarn install`. If `yarn` doesn't detect any changes in `package.json`, it won't install anything new and/or update `yarn.lock`

To update dependencies when you have lockfile present run:
```zsh
# for all dependencies

> yarn upgrade 
```
```zsh
# for a specific dependency

> yarn upgrade PACKAGE-NAME
```

Adding `--latest` flag at the end makes yarn ignore the specified version range and install the latest version(s).

## Maybe we should just deploy the lockfile alongside other files?

### tl;dr - no you don't (sometimes)

That depends on what your project is:
* Is your project an application? **Then: Yes**
* Is your project a library? **If so: No**

### Why should you care about lockfile maintenance for libraries?

There seems to be an agreement around committing the lockfile, both for applications and libraries. There's an [excelent post on yarnpkg](https://classic.yarnpkg.com/blog/2016/11/24/lockfiles-for-all/) covering this topic if you want to understand the reasoning behind it.

We'll focus on libraries as `Elements`, which we're working on, is one. Plus, committing the lockfile alongside the application pretty much solves the issue of unwanted updates.

### Handling lockfile in libraries

> **Important:** When you install dependencies in your application or library, only the top-level 
yarn.lock file is respected. Lockfiles within your dependencies will be ignored.

Because only the top-level lockfile is respected (the one form users project), `yarn` will look into the used library's `package.json` and install the packages with versions described there. Unless you pin each dependency to an exact version, users' projects might end up having different sub-dependencies depending on the time of installation.

So are we doomed? Kind of, users will always be the first people to discover a breaking change in a dependency (and hopefully file a bug report). To give you some perspective:

- your library dependencies have 20 sub-dependencies that can release new versions at any given moment
- your library is installed every hour
- people contribute to your library, let's say once a day

That means that 24 users can potentially run into bugs caused by dependency bumps in the same amount of time in which ONE contributor will (assuming that you don't commit `yarn.lock` to your library repo).

Although, because you don't want to force contributors to do a fresh install each time (it would allow discovering bugs earlier but at the same time discourage people from contributing at all), you need to make some kind of compromise. Once your library is released and well known you'll be just fine with users reporting bugs themselves, but before the first stable release, it's best to handle this yourself. In the end, you don't want people to give up your product, right?

##  Finding the right solution

Ok, so you've decided that you want to keep your dependencies up to date on a timely basis. What are the ways to do it?

### Dependabot

The first approach we took was `Dependabot` - a well-known tool for bumping dependencies. It checks for possible updates, opens Pull Requests with them, and allow users to review and merge (if you're confident enough with your test suite you can even set auto-merge)

We'd been already using Dependabot for security updates and it served the purpose really well!

Why did we decide not to go with it?

Unfortunately, it [misses](https://github.com/dependabot/dependabot-core/issues/2390) (at least at the time of writing this article) the ability to have duplicate updates for different `allow` types. That means you can't have e.g. daily updates for `dependencies` and weekly updates for `devDependencies` in the same project.

Also, as it turned out, later on, having new PR for each dependency update is a bit of PITA.

### Renovate

After figuring out that `Dependabot` does not fit our requirements, we've decided to look for alternatives. One of the most promising ones (and open-source!) was `Renovate`.

Even though the basic principle of bumping dependencies is the same, the tool itself seems very powerful and customizable. It has 3 applications (Github, Gitlab, and self-hosted), highly granular settings (you can even set custom rules for auto-merging of PR), and allows opening a PR for a batch of dependencies, instead of for each one.

As we are using GitHub for version control, the supported application for it was an obvious choice. Because though our usage was a bit unorthodox (updating only `yarn.lock` and not `package.json`), we wanted to test it on the self-hosted version, to avoid unnecessary PRs created by Renovate, or even worse - unwanted merges.

This is where we hit a wall with Renovate - even though it has a great range of options, we didn't manage to configure it the way we wanted - update **ONLY** `yarn.lock` once a week and create a single PR.

There are differences in configuring the self-hosted version and the GitHub application, which I don't think are documented well enough, and even though Renovate has a nice "onboarding PR" that shows you the possible outcome of your config, it's still really hard to get it right.

Because of that reason, we decided to not spend more time on publicly available solutions, and handle the lockfile maintenance ourselves.

### Your own CI job

You may ask: _"Why did you even bother with setting those dependency management systems? Isn't it easier to just run `yarn upgrade` on everything and call it a day?"_

And you would be partially right. The thing is that these systems probably do the exact same thing under the hood but put more attention to the possible failures and corner cases. And just because they are already battle-tested, we decided to check them first. Custom solutions built from scratch, in general, tend to be more fragile than the commercially available ones.

Since neither Dependabot nor Renovate met our needs at a time though, our way out was writing a custom CI job that:

1. Would bump dependencies for us
2. Run some basic tests against those changes
3. Create a PR
 
Our toolchain was:

- `CircleCI` for CI/CD
- `git` and `GitHub` for VCS
- `Yarn` as a package manager
- `Jest` for testing
- `Coffee®` for energy

#### **Custom command**

 ```bash
 ### bash

  $ git checkout main
  $ export BRANCH_NAME=feat/lockfile-maintenance-ci-job-$(date +"%m-%d-%Y") && git checkout -b $BRANCH_NAME
  $ yarn upgrade
  $ git add yarn.lock
  $ git commit -m "chore: weekly lockfile maintenance"
  $ git push --set-upstream origin $BRANCH_NAME
  $ BODY='{"head":''"'${BRANCH_NAME}'"'',"base":"main","title":"Weekly lockfile maintenance"}'
      && curl -X POST
      -H "Accept:application/vnd.github.v3+json"
      -u $GIT_AUTHOR_NAME:$GH_TOKEN https://api.github.com/repos/stoplightio/elements/pulls
      -d "$BODY"
 ```

The premise of this is:
1. Get the latest changes from main (no need to `git fetch` as this is being run in a fresh CI job each time) and create a feature branch with a name corresponding to the lockfile maintenance
```bash
  $ git checkout main

  $ export BRANCH_NAME=feat/lockfile-maintenance-ci-job-$(date +"%m-%d-%Y") && git checkout -b $BRANCH_NAME
```
2. Upgrade all of the dependencies in `yarn.lock` according to `package.json` - this mimics what happens for users upon a fresh install
```bash
  $ yarn upgrade
```
3. Push changes to remote
```bash
  $ git add yarn.lock
  $ git commit -m "chore: weekly lockfile maintenance"
  $ git push --set-upstream origin $BRANCH_NAME
```
4. Create a PR using GitHub API (more details in [GitHub API Documentation](https://docs.github.com/en/rest/reference/pulls))
```bash
  $ BODY='{"head":''"'${BRANCH_NAME}'"'',"base":"main","title":"Weekly lockfile maintenance"}'
      && curl -X POST
        -H "Accept:application/vnd.github.v3+json"
        -u $GIT_AUTHOR_NAME:$GH_TOKEN https://api.github.com/repos/stoplightio/elements/pulls
        -d "$BODY"
```

Both `$GIT_AUTHOR_NAME` and `$GH_TOKEN` are secrets from `CircleCI` - make sure you don't hard code your credentials in the CI config file and/or the command itself.

#### **CI configuration**
```yml
workflows:
  version: 2
  test-and-release:
    ...
  perform-lockfile-maintenance:
    triggers:
        - schedule:
            cron: "0 3 * * 1"
            filters:
              branches:
                only:
                  - main
    jobs:
      - lockfile-maintenance
```

Make sure you define the job as well:
```yml
jobs:
lockfile-maintenance:
    docker:
      - image: circleci/node:12
    steps:
      - checkout
      - run:
          command: | 
            ### THIS IS A PLACE FOR THE COMMAND FROM PREVIOUS PARAGRAPGH
```

By default, CircleCI runs workflows against all commits from all branches. This is definitely not the behavior we want to have for lockfile maintenance. The desired outcome is that it will run once a week against `main` branch. We also don't run any tests at this stage, as the PR created against the `main` branch will trigger the `test-and-release` workflow that is being run for each branch and contains a test suite, checks linting, and builds a project to see if there are no crashes.

That's where `cron` jobs come in handy. We first define that our `perform-lockfile-maintenance` workflow will be triggered by one (test yours using [this online tool](https://crontab.guru)) by putting cron job description in the `triggers/schedule` section. Then we apply an additional filter to it, so it only targets `main` at any given moment.

As for scheduling, we decided to go with Monday before work (Central European Time), so it is the first thing we look into at the beginning of the week. A contributor opens a PR containing changes made to `yarn.lock`, approves if it looks right, and merges the change to `main`.

And that's it! You've just set up your first lockfile maintenance flow!

## Possible improvements / aftermath

There are few more things you can do to improve your confidence even more:
- If you include examples of usage for your library like us (an integration for GatsbyJS, Angular, CRA) you can bump their dependencies as well. This is will assure that your library not only is properly tested internally but doesn't crash when applied to a real-life scenario
- Serve and an environment containing these integrations for each PR e.g. using Netlify. That will make the whole testing process much quicker as you won't need to check out the changes and run them locally on your own
- Strengthen your CI pipeline in general: the more is covered by your testing suite, the less you will have to check

## Summary

So there you go, we have just gone to a dependency hell and came back alive!

I believe that what I have described above will help you encounter fewer issues when developing your library, especially if you don't have a full team dedicated to testing bugs.

But even if I didn't convince you to do a weekly/monthly/whatever dependency bump I hope that this article gave you a strong understanding of the lockfile itself, why it is important when talking about compatibility across different machines, and seeing that lockfile maintenance does not have to be a terrible chore that takes an unreasonable amount of time.

If you feel like this article added some value to your current skillset though, please share it on social media and mention @11Sigma and @Stoplightio - we work very hard together to deliver great products, and one of them, Elements, was a basis for this article.