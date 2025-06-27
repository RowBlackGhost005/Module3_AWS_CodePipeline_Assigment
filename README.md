# CI/CD Pipeline with AWS Code Pipeline.

In this assigment I created a CI/CD Pipeline using AWS Code Pipeline to automate the deployment of a simple static HTML page.

# Requirements
An AWS Account

# Setup
For setting up this enviroment we are going to setup a repository in GitHub and a attach it to an AWS CodePipeline so it can grab the code from our repository and then deploying it to a S3 bucket (For this time due to this being a simple static HTML)

## Setup the repo
First we will be setting up a repo, in this case is this same exact repo, this contains a basic static HTML called `index.html`, this is what is going to be continously deployed on repo changes.

So setup a github repo and push some basic code to have as a base.

![Github Repository](doc/images/git-setup.png)

## Setup S3 Bucket
Now lets setup an S3 Bucket so we can store our 'built' code to be ready to deploy.

In this case because our app is a single static HTML what it will end up happening is that AWS Pipeline will take our HTML from Github and pass it to the S3 Bucket to deploy it.

So to do this in your AWS Dashbaord go to `S3` and click on `Create Bucket`

We will select a unique name.

![S3 Setup](doc/images/s3-setup.png)

Then scroll down to `Block Public Access Settings for this bucket` section and uncheck `Block All Public Access` and acknowledge the warning that the files might be public.

![S3 Allow Public Access](doc/images/s3-setup-access.png)

Then click `Create Bucket`

### Enable Static Web Hosting
Now that the bucket is setup, we will enable static web hosting so we can have access to our website once is deployed.

For this click on the bucket we just created in the S3 Dashboard and then go to `Properties` tab and scroll down to `Static website hosting` and click `Edit`

In this screen enable `Static Website hosting` and set the name of the `Index` document, in this case is `index.html`

![S3 Enable Hosting](doc/images/s3-enable-static.png)

For this basic setup of S3 web hosting these are all the configurations, now scroll down and click `Save Changes`.

### Enable Public Access
Now we will be granting public access to our S3 bucket objects so the users can be able to reach the site.

For this, in the bucket settings go to `Permissions` tab and go to `Bucket Policy` and click `Edit`.

In Edit Bucket Policy screen we will be adding the next JSON Permission:

```JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<S3NAME>/*"
    }
  ]
}
```
*Note: Change `<S3NAME>` with the name of your S3*

Now click on `Save Changes`
![S3 Permissions](doc/images/s3-permissions.png)

Now that we granted access we need to setup our S3 bucket to host the webpage, for this go to the `Properties` tab in our S3 bucket and scroll down to the `Static Website Hosting` here click `Edit`.

Now inthis screen `Enable` Static Web Hosting and select `Host a Static Website`.

Down below under `Index document` input the name of the html file that we are going to host, in this case is `Index.html`

Now there no more changes needed here, go ahead an click `Save Changes`
![Enable Web Hosting](doc/images/s3-enable-static.png)

Now our S3 bucket is ready to host our HTML Webpage and users will be able to access to it.

If you go back to the Static Website Hosting secction you now will see the URL to access the web page, save it for later once our CI/CD pipeline is ready to deploy changes.

## Setup AWS Code Pipeline
Now that everything around the pipeline is ready we will be setting it up all together.

Now navigate to your AWS Dashboard and search `Code Pipeline` and click on `Create Pipeline`

Here will be doing a series of steps to properly setup a pipeline that fit our needs.

In Choose Create Option screen, select `Build custom pipeline`

![Pipeline Step 1](doc/images/pipeline-s1.png)

Then in Choose Pipeline Settings Page, select a `name` for the pipeline.

In execution mode select `Queued`.

And then let AWS to create a role automatically for this pipeline.

![Pipeline Step 2](doc/images/pipeline-s2.png)

Now on Add Source Stage we will be setting a source provider, here select `GitHub via OAuth`

Go ahead and click `Connect to GitHub` and follow the steps to connect your account.

Once connected point at the repository we just created and select the branch if there are multiple of.

Then click `Next`
![Pipeline Step 3](doc/images/pipeline-s3.png)

On the Adding Build Stage we will be selecting Other build providers and select AWS Code Build.

Then on project name we will be creating a new Project Build, this is going to open a new window that we needto setup.

![Pipeline Step 4](doc/images/pipeline-s4.png)

### Create Build Project
On this screen we will be creating the Build Project, this we will be selecting a name.

On project type leave it on `Default Project`.

![Setup build project](doc/images/pipeline-s4-build.png)

Then scroll down to the `Buildspec` section, here select `Use a buildspec file` and input the name `buildspec.yml`

This buildspec file tells Code Build what to do to 'build' our project, so for this case we only need to tell it to sync data to our S3 bucket from GitHub.

To do this we will be adding a buildspec.yml file into our root folder in our GitHub repo with the following code:

Make sure the filename is `buildspec.yml` or that it matches whatever name you selected
```YAML
version: 0.2

phases:
  build:
    commands:
      - aws s3 sync . s3://BUCKETNAME --delete

artifacts:
  files:
    - '**/*'
```

The buildspec section should look like this:
![Setup buildspec](doc/images/pipeline-s4-buildspec.png)

Continue to scroll down leaving everything on their default settings and click on `Continue to Code Pipeline`.

Now with the Build Project integrated we can coninue leaving everything in here on default.
![Setup Build](doc/images/pipeline-s4-setup.png)

Now we will be `skipping` the Test Stage because our simple static webpage has noting else to be tested, so go ahead and click `Skip Test Stage`
![Setup Add Test](doc/images/pipeline-s5.png)

On the Add Deploy Stage screen we will setting up the deployment rules of our artifacts that were setup for creation in the 'build' stage, in this case basically is going to select our index.html and thats all.

Here we select `AmazonS3` as the deploy provider, make sure the region is in the same where your pipe line is.

In the Input Artifact section select `Build artifact`, this is telling that S3 is going to store whatever comes from build, and from build we specify that only our index.html is going through.

Then select a bucket based on the name.

And last select the `S3 Key` which is a file or folder where the project is going to be put, in this case just add `/` to deploy the files at the root of the bucket.

Then click `Next`
![Setup Deploy](doc/images/pipeline-s6.png)

In the last step we just need to make sure all of the settings are correctly set, here you can also review what we have done and if anything need changes you can go back in the step process.

If everything is OK, click on `Create Pipeline`

![Setup Review](doc/images/pipeline-s7.png)

Once finished the pipeline should start working automatically on the deployment on what is currently available in the Git Hub Repository.

![Pipeline Overview](doc/images/pipeline-overview.png)

Now, this first build might fail because we set it up in a way that in the 'Build' phase it will automatically replace the files from the GitHub repo to our S3 bucket, so now we need to add the permissions to that role.

## Setup Permissions for Pipeline
Now we will be granting access to our S3 Bucket to our Pipeline to write and delete files, to do this go to `IAM` in the AWS Dashboard.

Then go to `Roles` and look for the automatic role generated for the Code Build, it should be called something like `codebuild-HTML-CI-CD-service-role`, then click on it.
(Note that it need to start with `codebuild`)

Once in the settings, it should have one policy attached that was auto generated, lets edit this so we can add the policy directly to avoid creating a new one. So click on it

Here click on `Edit`.
![Policies Screen](doc/images/policies-mainscreen.png)

On the edit page select `JSON` to be able to see the policy in a JSON format and scroll down to the last policy object and add a `,`, then paste this policy:

*Note: Replace BUCKETNAME with the actual name of the bucket we created earlier*
```JSON
{
    "Effect": "Allow",
    "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
    ],
  "Resource": "arn:aws:s3:::BUCKETNAME/*"
}
```


It should end up looking like this, be mindful to keep the JSON format intact, and the click `Next`
![Policy Edit](doc/images/codebuild-policy-edit.png)

Now our CodeBuild section of our pipeline has access to our S3 bucket and its going to be able to put the files of our GitHub into the S3 bucket.

## Confirm Pipeline
Now that the permissions are in place, our pipeline is going to run smootly, now make some changes in the HTML file and push it to the Git Hub repo, it should automatically trigger a Deployment.

And here it is after one change, everything is setup ande deployed.
![Deployment Stage](doc/images/pipeline-working.png)


## Access The Website
Now that our website is deployed lets use the URL of the static website that we got earlier, if you don't have it go to:

(`S3 > Bucket > #Name#`), then Properties tab then Static Website Hosting, there should be a URL for accessing our website.

In my case is: http://ci-cd-bucket-csa.s3-website-us-east-1.amazonaws.com/

And if I use the browser to get to it, I get this:
![Website](doc/images/website-1.png)

So our website works and now its deployed completely automatically!

# Check Deployment of Changes
Lets modify the HTML to add another card and see if the page actually reflects the changes.

In my case these cards work using a JSON object, so all I need to do is add a new object and it should appear one more card.

![html edit](doc/images/html-edit.png)

I push the changes into github and the pipe line pick it up and started the process of deployment.

![Overview Pipeline](doc/images/pipeline-change-overview.png)

After a few moments I got the sucessfull message

![Overview Pipeline](doc/images/pipeline-change-finished.png)

And if I go to the website I see that in fact it got updated automatically based on my changes.

![Website](doc/images/website-2.png)

# Summary
By using AWS CodePipeline in integration with GitHub and setting up a CodeBuild project we are able to setup a basic CI/CD enviroment to push our changes to production directly from Github automatically, this saves time in the deployment process.

And due to how we setup this project is easy to 're-use' this setup for a more complex approach, like actually building something in the 'Build' stage instead of just copying files, this provies how this kind of workflows altough difficult to setup at times are able to save us a ton of time once setup correctly.

# Credits
Developed by: Luis Marin