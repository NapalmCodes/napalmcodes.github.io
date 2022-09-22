+++
author = "Shawn Vause"
title = "Getting Python to AWS Lambda From Non-Linux Environments"
date = "2021-12-03"
summary = """
Python dependencies are often platform specific and require compilation in the environments they will run within. AWS Lambda is not an easy environment to replicate locally. \
Using Docker we can install AWS Lambda platform compatible python dependencies on macOS and Windows. This allows developers to upload to the cloud from our local machines, thereby keeping feedback loops concise."""
tags = [
    "aws",
    "lambda",
    "python"
]
+++

AWS Lambda supports a variety of ways to get code and dependencies up and running in the cloud. You can use <a href="https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html" title="AWS Lambda Layers" target="_blank">Lambda layers</a>, <a href="https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html" title="AWS Lambda Custom Runtimes" target="_blank">custom runtimes</a> or for ultimate control just specify a full <a href="https://docs.aws.amazon.com/lambda/latest/dg/runtimes-images.html" title="AWS Lambda Container Support" target="_blank">container</a> definition and push that up to ECR/Docker Hub for use by the Lambda platform. However, as flexible as Lambda is, there are two approaches that stand above the rest as "drop-dead simple". You can write your code in the browser with the built in editor, but let's be honest that isn't the best development experience, or you can zip your assets up locally and push them to the cloud. This is of course provided that your assets/dependencies are less than the <a href="https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html" title="AWS Lambda Zip Size" target="_blank">50MB</a> cap at the time of writing this post. The remaining content of this post will focus on the zip deployment approach and problems you can run into when working on a Mac/Windows environment which most professional developers are likely to spend the bulk of their time in.
{{<preview_ph>}}

Python being the target function language for our routine has some design considerations to orient ourselves with when we do a ​pip install​ of package dependencies locally. Typically, a pip package will be compiled on the platform it is installed on by the python wheel. As a result, you can run into problems where MacOS does things differently than Windows or Linux during that compilation. For example, MacOS handles encryption differently than Windows and differently than Linux being a FreeBSD based operating system. The compiled dependency will likely be incompatible with the Linux OS running in the Lambda platform. Any zip file we push up to the cloud must have the dependencies compiled for the host Linux context!

So how do we solve the previously described problem? Enter the <a href="https://github.com/lambci/docker-lambda" title="lambci/docker-lambda Repository" target="_blank">lambci</a> collection of docker images which includes the ​<a href="https://github.com/lambci/docker-lambda/blob/master/python3.8/build/Dockerfile" title="lambci Python 3.8 Build Dockerfile" target="_blank">lambci/lambda:build-python3.8</a> for python version 3.8. This image includes all the necessary build tooling for installing python dependencies with pip. The goal of the lambci project is to produce a docker image that is as close to the AWS host environment for a Lambda function as possible. This is a great way to test your function code locally while iterating quickly during development. Using the build image previously mentioned, we are able to build our zip file within docker for uploading to AWS and share it out locally via a docker volume. Here is a handy script you can use in bash for creating your python code archive:

```bash
#!/bin/sh

# Execute from your python3.8 code directory

# Copy contents to temp directory
cp -r ./ ./temp && cd ./temp

# Run the lambci container with a volume in the personal working directory. Then execute the commands
# in the container to do a local pip install. Finally, zip contents from the /var/task path in the container.
docker run -v "$PWD":/var/task lambci/lambda:build-python3.8 cd /var/task && \
pip install --target ./packages -r requirements.txt && \
zip -r mycode.zip ./ -x "packages/*" && \ 
cd packages/ && \
zip -ur ../mycode.zip ./
```

The comments explain what is going on, but it is important to mention that when we create the zip file we initially exclude the packages directory created by the targeted pip install​. We then add the installed packages to the root of the zip avoiding folder nesting that will prevent the dependencies from being recognized by the Lambda runtime. Take a look at this <a href="https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-awscli.html" title="Using Lambda with the AWS CLI" target="_blank">documentation</a> to upload your zip into AWS Lambda via the AWS CLI! If all goes according to plan you will be running your function with properly detected dependencies!
