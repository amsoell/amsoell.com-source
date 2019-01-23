---
title: "Hosting static web sites with AWS"
date: 2019-01-07T11:43:58+01:00
tags: [ "devops", "aws", "s3", "codecommit", "codebuild", "hugo" ]

---

I've recently rebuilt my personal web site, and I wanted to take a moment to talk about the DevOps side of things for a change. I've used variety of platforms over the years, including many disparate technologies. Most recently the site was built leveraging [WordPress](https://wordpress.com), tracked via [GitHub](https://github.com), and deployed to a [DigitalOcean](https://www.digitalocean.com) instance using [DeployBot](https://deploybot.com). In the years since, Amazon's AWS offerings have continued to grow and I've become much more familiar with how they work—and how they all work together—so I wanted to take advantage of the opportunity here to see what it was like to go "all-in" on the AWS ecosystem on a project. To that end, this site is now entirely hosted, stored, tracked, built and deployed with AWS. Here's a bit of what that process was like, and what I've learned.

## CodeCommit

I'm a huge fan of Git and use it in my development workflow all day, every day. Most of my experience working with hosted Git platforms has been with [GitHub](https://github.com/amsoell) and [GitLab](https://gitlab.com/amsoell), but for this exercize I wanted to see what it was like to work entirely within the AWS platform, which meant using Amazon's [CodeCommit](https://aws.amazon.com/codecommit/) platform. I found it was very similar to these other platforms I've worked with in the past, and setting up a private repository to host my Hugo source was trivial: Navigate to the CodeCommit panel, click "Create Repository," and give it a repository name. From there, you get some very plain-English instructions for adding your remote and pushing your codebase.

{{< figure src="/images/posts/static_site_aws/codecommit_repository_setup.png" width="100%" >}}

*Cost*: Free (up to 5 active users, unlimited repositories, 50GB of storage and 10,000 Git requests each month)

## S3

Many developers know that AWS's S3 platform is a great tool to use to keep files for long-term storage or use as a content distribution network. I use it on most of my projects as a centralized file storage location, but it can also very capably host a static web site in its entirety. For my site, I'm using the Go-powered static site generator Hugo, so hosting directly in an S3 bucket makes perfect sense for a project like this. There are no server instances to provision, no application dependencies to install, just a simple file container.

This takes a couple of steps, but setting up an S3 bucket to serve a static web site is quite simple. First, navigate to the S3 panel and click the "Create bucket" button. Give it a bucket name to match your web site, and then click the "next" button twice to get to the *Set Permissions* panel. Uncheck all four boxes, which tells S3 that all files loaded into this bucket should be publicly readable, then click the "Create" button.

{{< figure src="/images/posts/static_site_aws/s3_bucket_setup_1.png" width="100%" >}}
{{< figure src="/images/posts/static_site_aws/s3_bucket_setup_2.png" width="100%" >}}

Next, you need to configure the bucket so that files added are *automatically* set to be publicly readable. Click into your new bucket and click on the "Permissions" tab, followed by the "Bucket Policy" button, and then paste in this configuration, substituting your own bucket name where it says BUCKETNAME:

{{< highlight json "hl_lines=9,linenostart=1" >}}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::BUCKETNAME/*"
        }
    ]
}
{{< / highlight >}}

{{< figure src="/images/posts/static_site_aws/s3_bucket_setup_3.png" width="100%" >}}

Once those settings are saved, there's one more step to make your bucket behave like an actual web server. Click on the "Properties" tab and select the "Disabled" button in the "Static website hosting" box. Select "Use this bucket to host a web site" and enter your default HTML document names. Make note of the **endpoint** address (this will be the URL for your site), then click "save."

{{< figure src="/images/posts/static_site_aws/s3_bucket_setup_4.png" width="100%" >}}

Now the bucket is all configured to host static web sites; Next we need set things up so that source code in CodeCommit can be built and placed into the bucket.

*Cost*: Depends on traffic, but for a low-access site less than $0.01 each month.

## CodeBuild

I've done a lot of work with continuous integration and deployment in the past, but mostly with tools like Circle CI and Travis. Sticking to the mission of working entirely within the AWS ecosystem, however, means digging into AWS CodeBuild. As the name suggests, CodeBuild can be used to perform build actions on your code and do something with the final result. There are a lot of configurable options here, but the process is actually pretty straight forward if you step through it carefully. And it all starts with a new configuration file in the project repository called *buildspec.yml*. Here's what mine looks like:

{{< highlight yaml "linenostart=1" >}}
version: 0.2

phases:
    install:
        commands:
            - wget -q -O libstdc++6 http://security.ubuntu.com/ubuntu/pool/main/g/gcc-5/libstdc++6_5.4.0-6ubuntu1~16.04.10_amd64.deb
            - sudo dpkg --force-all -i libstdc++6
            - wget -q https://github.com/gohugoio/hugo/releases/download/v0.53/hugo_extended_0.53_Linux-64bit.deb
            - dpkg -i hugo_extended_0.53_Linux-64bit.deb
    build:
        commands:
            - echo "Building web site"
            - hugo
artifacts:
    files:
      - '**/*'
    base-directory: 'public'
{{< / highlight >}}

Let's break this down into three sections. The **install phase** basically just installs all of the packages that we need in order to compile a Hugo site. In our case, we needed some additional C++ libraries and, of course, the Hugo package.

{{< highlight yaml "hl_lines=4-9,linenostart=1" >}}
version: 0.2

phases:
    install:
        commands:
            - wget -q -O libstdc++6 http://security.ubuntu.com/ubuntu/pool/main/g/gcc-5/libstdc++6_5.4.0-6ubuntu1~16.04.10_amd64.deb
            - sudo dpkg --force-all -i libstdc++6
            - wget -q https://github.com/gohugoio/hugo/releases/download/v0.53/hugo_extended_0.53_Linux-64bit.deb
            - dpkg -i hugo_extended_0.53_Linux-64bit.deb
    build:
        commands:
            - echo "Building web site"
            - hugo
artifacts:
    files:
      - '**/*'
    base-directory: 'public'
{{< / highlight >}}

The second section, listed under **build** simply runs the Hugo build command, which should result in the final site being built into a _public_ folder.

{{< highlight yaml "hl_lines=10-13,linenostart=1" >}}
version: 0.2

phases:
    install:
        commands:
            - wget -q -O libstdc++6 http://security.ubuntu.com/ubuntu/pool/main/g/gcc-5/libstdc++6_5.4.0-6ubuntu1~16.04.10_amd64.deb
            - sudo dpkg --force-all -i libstdc++6
            - wget -q https://github.com/gohugoio/hugo/releases/download/v0.53/hugo_extended_0.53_Linux-64bit.deb
            - dpkg -i hugo_extended_0.53_Linux-64bit.deb
    build:
        commands:
            - echo "Building web site"
            - hugo
artifacts:
    files:
      - '**/*'
    base-directory: 'public'
{{< / highlight >}}

Finally, under **artifacts**, we list out which files we treat as output. In our case it's _everything_ (denoted as "\*\*/\*") inside the "public" folder.

{{< highlight yaml "hl_lines=14-17,linenostart=1" >}}
version: 0.2

phases:
    install:
        commands:
            - wget -q -O libstdc++6 http://security.ubuntu.com/ubuntu/pool/main/g/gcc-5/libstdc++6_5.4.0-6ubuntu1~16.04.10_amd64.deb
            - sudo dpkg --force-all -i libstdc++6
            - wget -q https://github.com/gohugoio/hugo/releases/download/v0.53/hugo_extended_0.53_Linux-64bit.deb
            - dpkg -i hugo_extended_0.53_Linux-64bit.deb
    build:
        commands:
            - echo "Building web site"
            - hugo
artifacts:
    files:
      - '**/*'
    base-directory: 'public'
{{< / highlight >}}

Once this file is added to your Git repository, you will need to set up the actual build project in CodeBuild. Start by clicking the "Create build project." As I alluded to earlier, there are a lot of options here. Let's look a the important ones:

+ _Project name_: Just a friendly name to remind you what the build is for. I wrote mine to match the domain name.  
+ _Source provider_: Our code is coming from CodeCommit, so set that appropriately  
+ _Repository_: Should be set to the Git repository name we selected in CodeCommit  
+ _Environment image_: This defines what kind of computer is needed to build the site files. In our case we just want a standard Ubuntu machine; Keep this set to "Managed image" and we can pick one Amazon provides.  
+ _Operating system_: Ubuntu  
+ _Runtime_: Base (this means plain Ubuntu, with nothing extra)  
+ _Runtime version_: aws/codebuild/ubuntu-base:14.04  
+ _Service role_: We'll leave this as default, and let CodeBuild create a new service role with all the necessary permissions.  
+ _Build specifications_: Leave this one default as well, and it will automatically look for the **buildspec.yml** file we created.
+ _Artifacts type_: We want to store the output of the build process in S3, so select **Amazon S3**
+ _Bucket name_: Select the bucket we created earlier, in our case **amsoell.com**
+ _Name_: If you don't enter something here, the built site will be placed inside a folder in your S3 bucket. Set this to **/** to tell CodeBuild to put the build files directly in the bucket root.
+ _Remove artifact encryption_: Yes

That's a lot of options, but that should be all you need. Click the final "Create build project" and let's get ready to test this out.

*Cost*: Free (up to 100 build hours each month)

Now that our code is in AWS, we have a place to host the files and a mechanism by which we can compile our code into final site files, let's give this a trial run. If you're still in the CodeBuild page that lists out your build projects, click into the one you've just created and click "Start build." The next page will show you a quick summary of what it's going to do, and clicking on "Start build" again should kick things off. You can either watch the spinning **pending** message until it changes to **succeeded** or scroll down the page for a blow-by-blow of the build process. Either way, at the end should find your built site sitting quite comfortably in your S3 bucket, ready to be served up to the public.

## What's next?

So the code is versioned, compiled, and hosted—all for pennies a month. But we're only halfway there. Right now, the revision process is: check the changes back into CodeCommit, and then run the build through CodeBuild. But wouldn't it be great if one could trigger the other, and we could reduce this down to a single step? Additionally, we need to set this up with a custom domain name and SSL certificate, so while we're off to a great start we aren't quite where we want to be just yet. Check back for part two, where we'll see how we can accomplish off of this using AWS.
