---
title: Review pull requests in pre-production environments in Azure Static Web Apps
description: Learn how to use pre-production environments to review pull requests changes in Azure Static Web Apps.
services: static-web-apps
author: sinedied
ms.service: static-web-apps
ms.topic:  conceptual
ms.date: 05/08/2020
ms.author: yolasors
---

# Review pull requests in pre-production environments in Azure Static Web Apps

This article demonstrates how to use pre-production environments to review changes to applications deployed with [Azure Static Web Apps](overview.md).

A pre-production (staging) environment is a fully-functional staged version of your application that includes changes not available in production.

Azure Static Web Apps generates a GitHub Actions workflow in the repository. When a pull request is created against a branch that the workflow watches, the pre-production environment is built. The pre-production environment stages the app, enables you to perform reviews before pushing them to production.

Multiple pre-production environments can co-exist at the same time when using Azure Static Web Apps. Each time you create a pull request against the watched branch, a staged version with your changes is deployed to a distinct pre-production environment.

There are many benefits of using pre-production environments. For example, you can:

- Review visual changes between production and staging. For example, viewing updates to content and layout.
- Demonstrate the changes to your team.
- Compare different versions of your application.
- Validate changes using acceptance tests.
- Perform sanity checks before deploying to production.

> [!NOTE]
> Pull requests and pre-production environments are currently only supported in GitHub Actions deployments.

## Prerequisites

- An existing GitHub repository configured with Azure Static Web Apps. See [Building your first static app](getting-started.md) if you don't have one.

## Make a change

Begin by making a change in your repository. You can do it directly on GitHub as shown in the following steps.

1. Navigate to your project's repository on GitHub, then click on the **Branch** button to create a new branch.

    :::image type="content" source="./media/review-publish-pull-requests/create-branch.png" alt-text="Create new branch using GitHub interface":::

    Type a branch name and click on **Create branch**.

1. Go to your _app_ folder and change some text content. For example, you can change a title or paragraph. Once you found the file you want to edit, click on **Edit** to make the change.

    :::image type="content" source="./media/review-publish-pull-requests/edit-file.png" alt-text="Edit file button in GitHub interface":::

1. After you make the changes, click on **Commit changes** to commit your changes to the branch.

    :::image type="content" source="./media/review-publish-pull-requests/commit-changes.png" alt-text="Commit changes button in GitHub interface":::

## Create a pull request

Next, create a pull request from this change.

1. Open the **Pull request** tab of your project on GitHub:

    :::image type="content" source="./media/review-publish-pull-requests/tab.png" alt-text="Pull request tab in a GitHub repository":::

1. Click on the **Compare & pull request** button of your branch.

1. You can optionally fill in some details about your changes, then click on **Create pull request**.

    :::image type="content" source="./media/review-publish-pull-requests/open.png" alt-text="Pull request creation in GitHub":::

You can assign reviewers and add comments to discuss your changes if needed.

> [!NOTE]
> You can make multiple changes by pushing new commits to your branch. The pull request is then automatically updated to reflect all changes.

## Review changes

After the pull request is created, the [GitHub Actions](https://github.com/features/actions) deployment workflow runs and deploys your changes to a pre-production environment.

Once the workflow has completed building and deploying your app, the GitHub bot adds a comment to your pull request which contains the URL of the pre-production environment. You can click on this link to see your staged changes.

:::image type="content" source="./media/review-publish-pull-requests/bot-comment.png" alt-text="Pull request comment with the pre-production URL":::

Click on the generated URL to see the changes.

If you take a closer look at the URL, you can see that it's composed like this: `https://<SUBDOMAIN-PULL_REQUEST_ID>.<AZURE_REGION>.azurestaticapps.net`.

For a given pull request, the URL remains the same even if you push new updates. In addition to the URL staying constant, the same pre-production environment is reused for the life of the pull request.

To automate the review process with end-to-end testing, the [Azure Static Web App Deploy GitHub Action](https://github.com/Azure/static-web-apps-deploy) has the `static_web_app_url` output variable.
This URL can be referenced in the rest of your workflow to run your tests against the pre-production environment.

## Publish changes

Once changes are approved, you can publish your changes to production by merging the pull request.

Click on **Merge pull request**:

:::image type="content" source="./media/review-publish-pull-requests/merge.png" alt-text="Merge pull request button in GitHub interface":::

Merging copies your changes to the tracked branch (the "production" branch). Then, the deployment workflow starts on the tracked branch and the changes are live after your application has been rebuilt.

To verify the changes in production,  open your production URL to load the live version of the website.

## Limitations

- Staged versions of your application are currently accessible publicly by their URL, even if your GitHub repository is private.

    > [!WARNING]
    > Be careful when publishing sensitive content to staged versions, as access to pre-production environments are not restricted.

- The number of pre-production environments available for each app deployed with Static Web Apps depends on the [hosting plan](plans.md) you are using. For example, with the Free tier you can have 3 pre-production environments in addition to the production environment.

- Pre-production environments are not geo-distributed.

- Currently, only GitHub Actions deployments support pre-production environments.

## Next steps

> [!div class="nextstepaction"]
> [Setup a custom domain](custom-domain.md)
