# Build a CI/CD Pipeline on AWS

In this tutorial, you will set up a Continuous Integration and Delivery pipeline for a web application using various AWS services. 

We will cover:
* Website Hosting with Amazon S3
* Static content caching with Amazon CloudFront
* Version Control with AWS CodeCommit
* Build management with AWS CodePipelines
* and
* Build, Test, and Deployment with AWS CodeBuild

By the end of this tutorial, you'll have a static HTML website hosted in S3, as well as an automated workflow based on the AWS Code product family.

Workshop Length: Less than 2 hours

## Clone this Repository
Included in this repository are a few files that we'll use when setting up the pipeline. Before getting started, pull a copy of this repository to your machine with:

`git clone https://github.com/thomasphorton/ci-cd-pipeline-on-aws.git`

or download a zip of the [repository contents](https://github.com/thomasphorton/ci-cd-pipeline-on-aws/archive/master.zip).

## Host a Website with S3

### Create a new S3 Bucket
Click the 'Create bucket' button and enter a compliant bucket name, then click 'Next'. You can skip through the next group of options- they're all configurable later if you need them.

The next tab is for bucket permissions. You can leave all of these as default for now as well.

Review the settings and click 'Create bucket'. Note the bucket name as `<WEBSITE_BUCKET>`.

### Upload a file to the Bucket
Click the 'Upload' button, click 'Add files', and navigate to and then select `index.html` from this repository's `src` directory. Click 'Open' and then 'Upload'.

Right click the `index.html` file that you just uploaded, and select 'Open'. The file should open in a new tab. Notice the `X-Amz-Security-Token` in the querystring.

Try removing the querystring and loading the file. Because of the security settings on the bucket, S3 will return a 403 error.

Regardless of whether or not the file exists, S3 will return a 403 error instead of a 404. This is to prevent someone from snooping around.

### Set an S3 Bucket Policy
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

Open the Static website hosting pane again, and click the 'Endpoint link'. The site should open.

Click the 'This does not exist' link. This time, you should see a 404 error. When accessing the bucket through the S3 Website URL, errors work differently and you will only see 403s if the resource exists but the user does not have permission to access it.

## Create a Content Delivery Network (CDN) with CloudFront
At this point, we have a website hosted out of a single region. It's accessible worldwide, but a user on the other side of the world from our S3 bucket will still need to communicate all the way across to get content.

With CloudFront, you can stand up a global Content Delivery Network in just a few minutes. This allows users to access content from a location close to them, reducing latency.

### Creating a CloudFront Distribution
In the [CloudFront console](https://console.aws.amazon.com/cloudfront/), click 'Create Distribution'. Click the 'Get Started' button under 'Web'.

The 'Origin Domain Name' field will auto-populate with various resources from your account. You'll see the bucket you created earlier in this list- don't select it! Instead, enter the S3 Website URL.

For now, the rest of the settings can be left in their default configuration. Click 'Create Distribution'.

You should see your CloudFront distribution in the console with a status of 'In Progress'. The deployment can take a while, so we'll come back to it.

## Version Control your project with CodeCommit
Now that we have a way for people to access our website, we'll probably want to make some updates to it. If you're going to be making updates to a project, it's best to use some sort of Version Control to keep track of your changes.

### Configure Connection to CodeCommit
* [Connection Over HTTPS](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html?icmpid=docs_acc_console_connect#setting-up-https-unixes-account)
* [Connection Over SSH](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html)

### Create a CodeCommit Repository
In the [CodeCommit console](https://console.aws.amazon.com/codecommit), click the 'Create Repository' button. Provide a name and description, then click 'Create repository'. Note the name as `<REPO_NAME>`.

The next page is optional. Email notifications are helpful, but can be set up later so click 'Skip' for now.

### Add CodeCommit as a Git Remote
Now that our repository is ready, we need to add CodeCommit as a remote. You can do that with the following command:

```
git remote add codecommit ssh://git-codecommit.<REGION>.amazonaws.com/v1/repos/<REPO_NAME>
```

Once that finishes, push your branch to CodeCommit with

```
git push --set-upstream codecommit master
```

If successful, you should be able to navigate through the repo in the [CodeCommit console](https://console.aws.amazon.com/codecommit).

### Create a Pull Request
* Create a new branch

```
git checkout -b pr-example
```
* Open the repository in your IDE and make a change to `src/index.html`. Save the file.

* Commit the change.

```
git add -A && git commit -m "Change for pull request"
```

* Push the branch to your CodeCommit repository.

```
git push codecommit
```

* In the CodeCommit console, navigate to your repository.
* Click `Create pull request`
* Choose your `pr-example` branch as the Source.
* Click 'Compare'.
* CodeCommit should tell you that you the branch is mergeable. Add a Title and Description to your PR, and click 'Create'.

### Review the Pull Request
Open Pull Requests can be viewed in the 'Pull requests' section in the sidebar.

* Click the 'Changes' tab and review the git diff.
* Click the 'Commits' tab and review the git history for the Pull Request.
* Click the 'Merge' button.
* Confirm 'Merge' and that you'd like to delete the pull request branch.
* Your changes should now be visible in the 'master' branch of your repository.

Great! You've reviewed and merged a pull request. This still doesn't solve the problem of deploying your changes to the S3 bucket.

## Create a CI/CD Pipeline with CodePipeline and CodeBuild

### Look at our Buildspec
CodeBuild uses a file called a 'buildspec' to run commands. This repository contains a few different buildspecs that we will use.

* Open `buildspec-simple.yml` in your IDE.

A buildspec file is broken into different phases. In this simple example, the only phase we will use is the 'Build' phase.

The build phase uses the AWS CLI to sync between our repository's `./src` folder and the website's S3 bucket. Make a mental note- this will come up a little later.

### Create a CodeBuild Project
* https://aws.amazon.com/codebuild/

* Create a new S3 bucket for CodeBuild Artifacts
  * Bucket name: `<YOUR_ALIAS>-wdc-codebuild-artifacts`

* Create a new CodeBuild project using the following settings:

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

`Message: Error while executing command: aws s3 sync "./src" "s3://$WEBSITE_BUCKET". Reason: exit status 1`

We now know where the error occurred, but only have a vague message. Dive deeper by scrolling down to the 'Build logs'. Towards the end, you should see:

`fatal error: An error occurred (AccessDenied) when calling the ListObjects operation: Access Denied`

This means that the CodeBuild container does not have all of the permissions it needs to operate on AWS resources. Remember above when we were looking at our buildspec files? Each AWS CLI command will require a set of permissions to be granted. In this case, the `aws s3 sync` command needs to ListObjects in the bucket to keep everything in sync. The S3 sync command actually needs more than that- it needs to be able to PutObjects, DeleteObjects, and a few other things. Rather than guess-and-checking, we'll give CodeBuild full access over our WebsiteBucket for now.

### Granting Permissions to CodeBuild
CodeBuild containers are assigned IAM roles to delegate permissions. We'll need to create a policy to give permissions for our WebsiteBucket, and attach it to the CodeBuild IAM role.

#### Create an IAM Policy
* Navigate to the IAM dashboard in the AWS Console.
* Click 'Policies' in the sidebar, and then click 'Create policy'.
* Pick one of the following two options to create your policy:

##### Using the visual editor:
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

##### Using the JSON editor:
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

#### Attach the IAM Policy to the CodeBuild Role
* Navigate back to the CodeBuild project list in the AWS Console.
* Select the `wdc-site-build` project and select 'Update' from the 'Actions' dropdown.
* Scroll down to the 'Service role' section, and note the 'Role name'.
* Navigate to the IAM dashboard in the AWS Console.
* Click 'Roles' in the sidebar, and search for the CodeBuild role
* On the Role Summary page, click 'Attach policies' in the 'Permissions' tab.
* Search for `wdc-site-build-policy`, click the checkbox to select it, and click 'Attach Policy'.

#### Retry the Build
* Navigate back to the 

* Set up CodeBuild project
* Review buildspec file
* Run once


## CodePipeline
* https://aws.amazon.com/codepipeline/
