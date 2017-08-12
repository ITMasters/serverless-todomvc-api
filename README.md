# Serverless TodoMVC API
This readme accompanies the video provide to students of x on how to use the [Serverless Framework](https://serverless.com) to implement the [TodoMVC API Spec](https://github.com/tastejs/todomvc-api] on the AWS platform.

## Installation 
Instructions for installing Serverless can be found on the frameworks site [here](https://serverless.com). Effectively it is no different han installing any other Node.js, so you will need to install Node.js and npm first. How to do this is platform specific, there are installers provided for most major operating systems on the [Node.js download page](https://nodejs.org/en/) but my preferred solution is to use a package manager if one is available on your operating system. For OS X, you can follow [this guide](http://blog.teamtreehouse.com/install-node-js-npm-mac).

You can confirm Servless has been properly installed by running
> serverless --version

## Configuration
Serverless always you to easily deploy to multiple providers. In this tutorial we will be using AWS, but you could also use offerings from Google, Microsoft and IBM.

You need to create AWS Access Keys for Serverless to use. Instructions on how to do this can be found [here](https://serverless.com/framework/docs/providers/aws/guide/credentials/). Note that this guide tells you to create a new user with admin access, which is not best practice but as the page notes 
>Unfortunately, the Framework's functionality is growing so fast, we can't yet offer you a finite set of permissions it needs (we're working on this). Consider using a separate AWS account in the interim, if you cannot get permission to your organization's primary AWS accounts.

I recommend creating a new Serverless profile for this tutorial, which can be done by running 
> serverless config credentials --provider aws --key aws-key --secret aws-secret --profile todomvc-api
where aws-key and aws-secret are your AWS access key and secret respectively.

## Using this repository
This repository contains a completed version of the TodoMVC API. To try it out for yourself, clone this repostitory, change into it's directory and then run
> serverless deploy
You should see a url for the http endpoints, which you can copy replace the `api_url` at the top of the app.js file.

If you then open index.html you should have a functional todo appi powered by your very own serverless api!
