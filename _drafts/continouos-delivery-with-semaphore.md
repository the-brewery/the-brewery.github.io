---
layout:     post
title:      Continuous Delivery with Semaphore
tags:       howtos continuous-delivery semaphore
categories: howtos
author:     Olof Montin
---
The Continuos Delivery is about automating our deployments and track them with specific commits. We're able to automaticly deploy to our staging environments whenever someone pushes anything that's passing the tests. And when deploying to production, we'll have a list of commits that we know will work. We're also able to do rollbacks whenever we find any deployments that fails.

We're using [Semaphore](https://semaphoreci.com) for this at the moment, which is fairly easy to setup. It mainly consists of three steps:

1. Choose a repository to test and deploy from.
2. Add some lines of _bash_ that tests the application (`npm install && npm test` for example).
3. Add a server to deploy to with some _bash_ lines that makes the actual deployment (`npm run build && rsync -az build/ user@host:app` for example)

[Semaphore](https://semaphoreci.com) do even have some ready-made recipies for deploying to an AWS S3 bucket, Heroku and some more services. These recipies will generate some environment variables for you and some lines of _bash_ with the _[aws-cli](https://aws.amazon.com/cli/)_ for example. Which you'll be able to extend with more stuff if needed.

The downside of using [Semaphore](https://semaphoreci.com) is that all testing and deployment settings are stored at [Semaphore](https://semaphoreci.com). Preferable, we want everything to be stored as configuration files and scripts in the repository (like [Bitbucket's Pipilines](https://bitbucket.org/product/features/pipelines) or [Wercker](http://wercker.com/)). Although, as a workaround, we can store all the testing and deployment lines of _bash_ as scripts in our repositories.

Sounds great, but how?
----------------------

1. Click the _add project_ button at [Semaphore](https://semaphoreci.com).
2. Then you'll be able to choose a repository to use.
3. Select branch - preferably _master_.
4. [Semaphore](https://semaphoreci.com) will then analyze your repository and based on your code it'll give you some standard setup with dependency installation and a test job. Which will be something like:
  * PHP: `composer install --prefer-source` and `phpunit`
  * Node: `npm install` and `npm test`
  * Python: `python setup.py test`
5. It's great if you can follow some standard way of executing your tests, even when things are getting a bit complex with combinations of Wordpress themes and Angular modules in the same repository.
6. Mock your tests against data sources. Although, there are several databases installed on the test machines that will run your tests.

### Deployment

When the tests are setup, you'll be able to add servers for deployment. The term _server_ is not just a server, it's practically a named collection of some lines of _bash_. So you can deploy to several instances, S3 buckets and docker repositories for the same _server_.

[Semaphore](https://semaphoreci.com) runs all tests and deployments on dedicated virtual machines. They're fired up with Ubuntu 14.04 LTS 64-bit and some standard setup of packages. Usually they come in a clean state, which means that your application needs to install it's dependencies again when deploying.

Deployments are fairly easy, it's usually about installing dependencies, build your app and then sync you build to some remote host. A thing to solve though when automating is to handle _ssh-keys_. [Semaphore](https://semaphoreci.com) will generate a private _ssh-key_ for you when adding a _server_. In order to use that key, you must generate a public key from it and place that public key in the `.ssh/authorized_keys` file at the host.

Generate a public key like this:

```bash
# Generate the public key
ssh-keygen -y -f semaphore-private-key > semaphore.pub
# Upload it to the server
gzip -c semaphore.pub | ssh user@host 'gunzip -c >> .ssh/authorized_keys'
```

When deploying, the Semaphore test machine will certainly not have the host's ssh key in the `.ssh/known_hosts` file. Therefore you'll need to add a line for that before doing any ssh commands, liks rsync for example.

```bash
ssh-keyscan -H -p 22 host >> ~/.ssh/known_hosts
```

So, here's some examples of deployments:

* Server-side Node application:

  ```bash
  # Install dependencies
  npm install
  # Sync 
  ssh-keyscan -H -p 22 host >> ~/.ssh/known_hosts
  rsync -az --exclude=.git . user@host:app
  # Reload your node app if using pm2
  ssh user@host 'bash --login -c "pm2 reload app/pm2-process.json"'
  ```

* Client-side Node application:

  ```bash
  # Install dependencies
  npm install
  # Build the application
  npm run build
  # Upload to some S3 bucket
  aws s3 sync $S3_DIRECTORY s3://$S3_BUCKET_NAME/ --acl=public-read --delete --exclude '.git/*'
  ```

When to deploy
--------------

When we're now able to fully automate the deployments we can deploy imediately after each successful test. At least to some staging environment. Although, there are some cases we've come across when this is not acceptable. For example: when we're deploying _test-releases_ for a some test-group to manually start testing against. In these cases we can't automaticly deploy new features in the middle of a test-phase. Therefor we're adding another deploy environment and end up with three: development with internal testing and review with automatic deployments; testing within some regular test-release-cycles; and then production.
