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

Review the settings and click 'Create bucket'.

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
In the [CodeCommit console](https://console.aws.amazon.com/codecommit), click the 'Create Repository' button. Provide a name and description, then click 'Create repository'.

The next page is optional. Email notifications are helpful, but we can set those up later so click 'Skip'.

If you cloned this repository, skip to 'Add CodeCommit as a Git Remote'.

If you downloaded the .zip, you'll need to initialize a git repo. To do that, navigate to the folder containing `ci-cd-pipeline-on-aws-master.zip` in your terminal and unzip it with

```
unzip ci-cd-pipeline-on-aws-master.zip
```

Move to the directory with

```
cd ci-cd-pipeline-on-aws-master.zip
```

and initialize a git repo with

```
git init
```

Create an initial commit with

```
git add -A && git commit -m "Initial commit"
```

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

## CodeCommit
* How do you keep track of changes?
* How do you know you've got the right version?
* https://aws.amazon.com/codecommit/
* Explain version control, git repository
* Clone our git repository from GitHub
* Create CodeCommit repository
* explain SNS
* Add CodeCommit repo as a remote
* Create a branch
* Make a change
* Push to CodeCommit
* Create Pull Request in console
* Merge PR
* Update master branch
* Update in S3
    * Still a manual process!


## CodeBuild
* https://aws.amazon.com/codebuild/

* Set up CodeBuild project
* Review buildspec file
* Run once


## CodePipeline
* https://aws.amazon.com/codepipeline/
