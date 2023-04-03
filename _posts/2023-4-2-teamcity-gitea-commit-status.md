---
layout: post
title: Posting Gitea commit status in a TeamCity build
---

TeamCity + Gitea + C# scripts = ❤️ Commit status

TeamCity and Gitea are well-known as useful tools for building self-hosted CI/CD, but one of the main constraints for many engineers is that one cannot publish commit status from TeamCity to Gitea out of the box. However, the clue is that TeamCity has C# scripts runner and a meta-runner feature, whereby it is possible to create reusable build steps for commit status publishing with Gitea API. 

### Prerequisites

1. Gitea server
2. Gitea repo where commit status will be published
3. TeamCity server with C# script runner

### Scheme of interaction
1. The first build step will clone the Gitea repo with C# scripts and publish a commit status that the build was started.
2. The second step will run when the build is finished or in case of an error, so it will post the following build result: “success” or “error” with error details.

### C# scripts to post commit status 
Gitea has a simple [REST API](https://try.gitea.io/api/swagger#/repository/repoCreateStatus), which can be used via common HttpClient. API of TeamCity a little bit more complicated, so [TeamCitySharp](https://github.com/mavezeau/TeamCitySharp) NuGet package can be utilized.

<details>
<summary>PublishCommitStatusStart.csx</summary>
<script src="https://gist.github.com/abagrov/142e036bfaa900ce5d48b52a56261832.js"></script>
</details>

<details>
<summary>PublishCommitStatusEnd.csx</summary>
<script src="https://gist.github.com/abagrov/0ac4c46e5e12aea0b455bb1c85fa5c62.js"></script>
</details>

You can fork my [repo with scripts](https://github.com/abagrov/TeamCityGiteaCommitStatus) and make changes.

#### Steps to create meta-runner build step

TeamCity has a meta-runner feature, which can create reusable build steps. So you can define multiple build steps and combine them in one build step with parameters, and use this build step in builds. In TeamCity navigate to Administration - Root Project - Meta-Runners.
You will need to create two meta runners - `PublishGiteaCommitStatusStart` and `PublishGiteaCommitStatusEnd`. 

First meta-runner `PublishGiteaCommitStatusStart` will clone my [repo with scripts](https://github.com/abagrov/TeamCityGiteaCommitStatus) to TeamCity work directory and run `PublishCommitStatusStart.csx`.

Second meta-runner `PublishGiteaCommitStatusEnd` will query TeamCity for build status, and publish actual build result. 

![Added two meta-runner](/images/teamcity-gitea-commit-status/metarunners.png)
<script src="https://gist.github.com/abagrov/d1cdb8ef6c6fc0b242a0e6ecc7f36829.js"></script>
<script src="https://gist.github.com/abagrov/e86f19c9a3e3dab88b10498466cb6c4f.js"></script>

Parameters, that likely will be the same in every project, can be filled with your own in meta-runner XML definition (or you will have to specify them every build step):
1. `GITEA_URL` - URL of your Gitea server.
2. `REPO_OWNER` - owner of Gitea repo.

These parameters can be overridden when you will be creating build step.

Then you need obtain authorization token for Gitea to publish commit status. In Gitea, click on your profile icon - Settings - Applications tab - Generate token.

After that, you need to create configuration parameter to store this token. In TeamCity navigate to Root project - Parameters - Add new Parameter. Name it `GITEA_TOKEN_COMMIT_STATUS`, set type = password, hidden and read-only.

### Add publish commit status build step to your build

After all, you can add publish commit status build steps like any other built-in steps.


![New build step available](/images/teamcity-gitea-commit-status/addPublishCommitStatus.png)

Setup publish commit start build step.
![Publish Gitea commit status start build step](/images/teamcity-gitea-commit-status/publishGiteaCommitStatusStart.png)

Then setup publish commit end build step.
![Publish Gitea commit status end build step](/images/teamcity-gitea-commit-status/publishGiteaCommitStatusEnd.png)

It is important to note that the second step, called `Publish Gitea commit status end` must be configured with the following condition: `Execute step: Always, even if build stop command was issued`.
