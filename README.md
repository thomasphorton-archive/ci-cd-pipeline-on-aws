# Build a CI/CD Pipeline on AWS

In this workshop, you will set up a Continuous Integration and Delivery pipeline for a web application using various AWS services. 

We will cover:
* Website Hosting with Amazon S3
* Static content caching with Amazon CloudFront
* Version Control with AWS CodeCommit
* Build, Test, and Deployment with AWS CodeBuild
*and*
* Build automation with AWS CodePipelines

By the end of this tutorial, you will have created a static HTML website hosted in S3, as well as an automated workflow based on the AWS Code product family.

Workshop Length: Less than 2 hours

## Clone this Repository
Included in this repository are a few files that we'll use when setting up the pipeline. Before getting started, pull a copy of this repository to your machine with:

`git clone https://github.com/thomasphorton/ci-cd-pipeline-on-aws.git`

or download a zip of the [repository contents](https://github.com/thomasphorton/ci-cd-pipeline-on-aws/archive/master.zip).

## Hosting a Website with S3
### Create a new S3 Bucket
Navigate to the S3 dashboard in the AWS Console, click the 'Create bucket' button and enter a compliant bucket name, then click 'Next'. You can skip through the next group of options- they're all configurable later if you need them.

The next tab is for bucket permissions. You can leave all of these as default for now as well.

Review the settings and click 'Create bucket'. Note the bucket name as `<WEBSITE_BUCKET>`.

### Upload a file to the Bucket
Click the 'Upload' button, click 'Add files', and navigate to and then select `index.html` from this repository's `src` directory. Click 'Open' and then 'Upload'.

Right click the `index.html` file that you just uploaded, and select 'Open'. The file should open in a new tab. Notice the `X-Amz-Security-Token` in the querystring.

Try removing the querystring and loading the file. Because of the security settings on the bucket, S3 will return a 403 error.

Regardless of whether or not the file exists, S3 will return a 403 error instead of a 404. This is to prevent someone from snooping around.

### Set an S3 Bucket Policy
**WARNING:** If you are using a Burner account to do this tutorial, you should modify a few steps in this section. Allowing public access to an S3 bucket in a burner account will lead to account closure within an hour or so. To avoid this, skip ahead to 'Create a Content Delivery Network (CDN) with CloudFront'.

Before configuring the S3-hosted website, you'll need to set a policy that allows users to read objects from your bucket. For more detailed information, check out the docs page for [bucket policies for S3 Websites](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteAccessPermissionsReqd.html).

To add a policy, in the 'Permissions' tab of your bucket, click the 'Bucket Policy' button.

In the Bucket policy editor, add the following policy and click 'Save'. Make sure to swap out <BUCKET_NAME> for your bucket name!

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>/*"
            ]
        }
    ]
}
```

If you've done this right, you should see a warning that the bucket now has public access. Refresh `index.html`, and you should see the file.

Try clicking the 'This does not exist!' link. In the browser console, you should see that S3 is returning a 403 error.

### Configure S3 static site
In the Bucket Properties tab, select 'Static website hosting'. Select the radio button for 'Use this bucket to host a website', and enter 'index.html' as the index document. Click 'Save'.

Open the Static website hosting pane again, and click the 'Endpoint link'. The site should open. Note the URL as the `<S3_WEBSITE_URL>`, we'll use it later.

Click the 'This does not exist' link. This time, you should see a 404 error. When accessing the bucket through the S3 Website URL, errors work differently and you will only see 403s if the resource exists but the user does not have permission to access it.

## Create a Content Delivery Network (CDN) with CloudFront
At this point, we have a website hosted out of a single region. It's accessible worldwide, but a user on the other side of the world from our S3 bucket will still need to communicate all the way across to get content.

With CloudFront, you can stand up a global Content Delivery Network in just a few minutes. This allows users to access content from a location close to them, reducing latency.

### Creating a CloudFront Distribution
In the [CloudFront console](https://console.aws.amazon.com/cloudfront/), click 'Create Distribution'. Click the 'Get Started' button under 'Web'.

The 'Origin Domain Name' field will auto-populate with various resources from your account.

If you set up S3 Website Hosting in the previous steps, ignore the S3 bucket that was pre-populated and instead enter the bucket's S3 Website URL (noted earlier as `<S3_WEBSITE_URL>`).

If you are skipping the S3 Website Hosting section, select the S3 bucket from the dropdown. For the purposes of this workshop, the difference is negligible.

The rest of the settings can be left in their default configuration. Click 'Create Distribution'.

You should see your CloudFront distribution in the console with a status of 'In Progress'. The deployment can take a while, so move ahead to the next section and check back in 20 minutes.

### After the CloudFront Deployment has Finished
1. Click the ID of the distribution to view its details.

1. Copy the 'Domain Name' and paste it in your browser's navigation bar.
  * If you did not set up S3 Website Hosting, append `/index.html` to the domain name.

You are now accessing files from the CDN. The files will be served from the closest edge location, and cached based on the distribution settings.

### Invalidate Cached Objects
Sometimes you can't wait for objects to expire from the cache. To force the CDN to fetch new objects, you need to create an Invalidation.

To see invalidations in action:
1. View a file from the CDN (i.e. `index.html`).

1. Change the file in some way. Depending on where you are in the other sections of this workshop, it can be a manual change to an S3 object, or a Pipeline deployment.

1. View the file again in the CDN. You should still see the old version.

1. In the details page of your CloudFront Distribution, click the 'Invalidation' tab and then click 'Create Invalidation'.

1. The dialogue shows a few example patterns for object paths. For this exercise, enter `*` to invalidate all objects in the cache.

1. Click 'Invalidate'.

1. The invalidation Status should show 'In Progress', and will complete in a few minutes.

1. View the file from the CDN again. You should see the new version.

## Version Control your project with CodeCommit
Now that we have a way for people to access our website, we'll probably want to make some updates to it. If you're going to be making updates to a project, it's best to use some sort of Version Control to keep track of your changes.

### Configure Connection to CodeCommit
Before using AWS CodeCommit, you'll have to do some configuration to make it possible to push to your repository. You have two options- I prefer to connect with SSH.
* [Setup Steps for Connection Over SSH](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html)
* [Setup Steps for Connection Over HTTPS](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html?icmpid=docs_acc_console_connect#setting-up-https-unixes-account)

Windows users- I haven't fully tested this out. This [StackOverflow question about SSH Keys on Windows 10](https://stackoverflow.com/questions/31813080/windows-10-ssh-keys) might be useful.

### Create a CodeCommit Repository
In the [CodeCommit console](https://console.aws.amazon.com/codecommit), click the 'Create Repository' button. Provide a name and description, then click 'Create repository'. Note the name as `<REPO_NAME>`.

#### Configure email notifications
This next page is where you can assign an SNS topic to receive notifications for repository events. An SNS topic can be configured to fan out to many things including email, SMS, and Chime webhooks!

Notifications are optional and can be set up later so click 'Skip' for now.

### Add CodeCommit as a Git Remote
Now that our repository is ready, we need to add CodeCommit as a remote. You can do that with the following command:

```
git remote add codecommit ssh://git-codecommit.<REGION>.amazonaws.com/v1/repos/<REPO_NAME>
```

Once that finishes, push your branch to CodeCommit with

```
git push --set-upstream codecommit master
```

If successful, you should be able to navigate through the repo and view files in the [CodeCommit console](https://console.aws.amazon.com/codecommit).

### Create a Pull Request
Create a new branch:

```
git checkout -b pr-example
```

Open the repository in your IDE and make a change to `src/index.html`. Save the file.

Commit the change:

```
git add -A && git commit -m "Change for pull request"
```

Push the branch to your CodeCommit repository:

```
git push codecommit
```

In the CodeCommit dashboard:
1. Navigate to your repository.

1. Click 'Create pull request'

1. Choose your `pr-example` branch as the Source.

1. Click 'Compare'.

1. CodeCommit should tell you that you the branch is mergeable. Add a Title and Description to your PR, and click 'Create'.

### Review the Pull Request
Open Pull Requests can be viewed in the 'Pull requests' section in the sidebar.

1. Click the 'Changes' tab and review the git diff.

1. Click the 'Commits' tab and review the git history for the Pull Request.

1. Click the 'Merge' button.

1. Confirm 'Merge' and that you'd like to delete the pull request branch.

1. Your changes should now be visible in the 'master' branch of your repository.

Great! You've reviewed and merged a pull request. The next step is to figure out a way to automate deploying the changes.

## Create a CI/CD Pipeline with CodePipeline and CodeBuild

### Buildspec Files
CodeBuild uses a YAML file called a 'Buildspec' to manage the build process. This repository contains a few different buildspecs that we will use.

Open `./buildspec-simple.yml` in your IDE.

A buildspec file is broken into different sections called phases. In this simple example, the only phase we will use is 'Build'.

The build phase uses the AWS CLI to sync between our repository's `./src` folder and the website's S3 bucket. Make a mental note- this will come up a little later.

### Create a CodeBuild Project
* https://aws.amazon.com/codebuild/

Before setting up the CodeBuild Project, create a new S3 bucket for CodeBuild Artifacts

  * Bucket name: `<YOUR_ALIAS>-wdc-codebuild-artifacts`

Once that's done, head over to the CodeBuild dashboard and create a new Project using the following settings:

#### Project Settings
* Project Name: `wdc-site-build`

#### Source
* Source Provider: `AWS CodeCommit`
* Repository: `<REPO_NAME>`

#### Environment
* Environment Image: image managed by AWS CodeBuild
* Operating system: `Ubuntu`
* Runtime: `Node.js`
* Runtime version: `aws/codebuild/nodejs:10.1.0`
* Build specification: Use the buildspec.yml in the source code root directory
* Buildspec name: `buildspec-simple.yml`

#### Artifacts
* Type: `Amazon S3`
* Name: `/`
* Path: `wdc-site-build`
* Namespace type: `Build ID`
* Bucket name: Select S3 bucket created at the beginning of this section.

#### Service role
* Create a service role in your account
* Role name: leave default

#### Show Advanced Settings -> Environment Variables
We'll want to pass some information to CodeBuild about our environment to use when executing the build:

* Name: WEBSITE_BUCKET
* Value: `<WEBSITE_BUCKET>`

Click continue, review the settings, and click 'Save and Build'.

### Start a New Build
The next page is where you will kick off a build. The only thing you need to do here is to select the repository branch to use in the build.

* Branch: `master`

Click `Start build`.

### Diagnosing Build Failures
Pretty quickly, you should see that the build status has changed to `Failed`.

On the build page, scroll down to 'Phase details' to see a status table for each of the phases. Click the arrow next to 'BUILD' to expand the details and view the error message.

```
Message: Error while executing command: aws s3 sync "./src" "s3://$WEBSITE_BUCKET". Reason: exit status 1
```

We now know where the error occurred, but only have a vague message. Dive deeper by scrolling down to the 'Build logs'. Towards the end, you should see:

```
fatal error: An error occurred (AccessDenied) when calling the ListObjects operation: Access Denied
```

This means that the CodeBuild container does not have all of the permissions it needs to operate on AWS resources.

Remember above when we were looking at our buildspec file? Each AWS CLI command will require a set of permissions to be granted. In this case, the `aws s3 sync` command needs to ListObjects in the bucket to keep everything in sync. The S3 sync command actually needs more than that- it needs to be able to PutObjects, DeleteObjects, and a few other things.

Rather than guessing-and-checking, we'll give CodeBuild full access over our WebsiteBucket for now.

### Granting Permissions to CodeBuild with an IAM Policy
CodeBuild containers are assigned IAM roles to delegate permissions to access AWS resources. We'll need to create an IAM policy to give permissions for our WebsiteBucket, and attach it to the CodeBuild IAM role.

1. Navigate to the IAM dashboard in the AWS Console.

1. Click 'Policies' in the sidebar, and then click 'Create policy'.

1. Pick one of the following two options to create your policy:

#### Using the Visual Editor:
* Service: S3
* Actions: All S3 actions
* Resources:
  * Specific
  * bucket -> Add ARN
    * Bucket name: `<WEBSITE_BUCKET>`
  * object -> Add ARN(s) ->
    * Bucket name: `<WEBSITE_BUCKET>`
    * Object name: `*` (Checkbox for 'Any')
* Click 'Review policy'
* Name your policy `wdc-site-build-policy` and click 'Create Policy'

#### Using the JSON Editor:
1. Paste the following JSON into the text area:

  ```
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "WebsiteBucketFullAccess",
        "Effect": "Allow",
        "Action": "s3:*",
        "Resource": [
            "arn:aws:s3:::<WEBSITE_BUCKET>/*",
            "arn:aws:s3:::<WEBSITE_BUCKET>"
        ]
      }
    ]
  }
  ```

1. Click 'Review Policy'
1. Name your policy `wdc-site-build-policy` and click 'Create Policy'

While it is possible to directly write this policy into the IAM Role, separating the policy out will allow you to attach it to additional IAM resources as needed. This can come in handy for debugging as your projects grow in complexity.

#### Attach the IAM Policy to the CodeBuild Role
1. Navigate back to the CodeBuild project list in the AWS Console.

1. Select the `wdc-site-build` project and select 'Update' from the 'Actions' dropdown.

1. Scroll down to the 'Service role' section, and note the 'Role name'.

1. Navigate to the IAM dashboard in the AWS Console.

1. Click 'Roles' in the sidebar, and search for the CodeBuild role

1. On the Role Summary page, click 'Attach policies' in the 'Permissions' tab.

1. Search for `wdc-site-build-policy`, click the checkbox to select it, and click 'Attach Policy'.

#### Retry the Build
1. Go back to the CodeBuild dashboard.

1. Click 'Build history' in the sidebar.

1. Select the latest run and click 'Retry'.

1. After a minute or two, the status of the new run should change to 'Succeeded'.

1. If you've made any changes to the files in `./src`, they should be visible in the S3 bucket.

## Automate Your Builds with CodePipeline
* https://aws.amazon.com/codepipeline/

1. Go to the AWS CodePipeline dashboard in the AWS Console.

1. Click 'Create Pipeline'.
  * Pipeline name: `wdc-site-pipeline`.


1. Click 'Next step'.

1. Configure as follows:

### Source Location
* Source provider: AWS CodeCommit
* Repository name: `<REPO_NAME>`
* Branch name:

### Build
* Build provider: AWS CodeBuild
* Select an existing build project
* Project name: `wdc-site-build`

### Deploy
* Deployment provider: No Deployment

### AWS Service Role
* Role name:
  * Click 'Create role'
  * Click 'Allow'
  * Role name will be populated

Review the pipeline settings and click 'Create Pipeline'. The pipeline will run using the most recent commit in your repository.

## Leveling Up Your Build
At this point, our build process is a simple AWS CLI command to copy a directory into an S3 bucket. This is a great proof of concept, but it's hardly ever that simple.

If you finish the workshop early, try building a more complex pipeline!

Ideas:
* Build a react app
* Run unit tests
* Deploy to a 'Gamma' stage
* Add a manual approval step to your pipeline
