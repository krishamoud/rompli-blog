---
title: "Running Meteor and Mongo in a Single Docker Container with Supervisord"
date: 2017-07-21T05:03:27-04:00
subtitle: ""
tags: ["meteor", "docker", "supervisord", "mongo"]
---

The goal of this post is to show you how to run your meteor.js application with MongoDB in one Docker container.  

**_Note:_** This should only be done for testing or prototypes.  

### Step 1: Get the example app

This tutorial was written as of `meteor 1.5.1`
```
git clone https://github.com/meteor/todos mongo-meteor
cd mongo-meteor
meteor npm install
meteor update
```

**_Note:_** If you plan to run the application on `localhost` then you'll have to remove the `app-prod-security` package

```
# .meteor/packages
...

# production
juliancwirko:postcss
standard-minifier-js@2.1.1
ddp-rate-limiter@1.0.7
# app-prod-security
```

### Step 2: Build and extract the application

We're building the application for Linux, and I'm going to deploy it to Rompli.
```
meteor build --architecture=os.linux.x86_64 --server https://todos.rompliapp.com:443 ../output
cd ../output
tar -zxvf mongo-meteor.tar.gz
```

### Step 3: Supervisord

For us to run meteor and mongo in the same container, we have to use [supervisord](http://supervisord.org/).

We need to create a `supervisord.conf` file to run `mongod` first, then run `node main.js` which is the entry point for the built meteor app.

```
cd bundle
touch supervisord.conf
```

Edit your newly created `supervisord.conf` file to look something like this:

```
[supervisord]
nodaemon=true

[program:mongod]
command=/usr/bin/mongod
autorestart=true

[program:meteor]
directory=/bundle
command=node main.js
environment=
          MONGO_URL=mongodb://localhost:27017/test,
          ROOT_URL=https://localhost
autorestart=true
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
```

### Step 4: The Dockerfile

I created a base image using `alpine:edge` that has `node 4.8.4`, `mongo`, and `supervisord` already installed and it's `224MB`.  If you can make a smaller image, please let me know how.  The gist for this image can be found [here](https://gist.github.com/krishamoud/2b8a21f1c9082b1ef557118343fad698).

First, create the Dockerfile.

```
touch Dockerfile
```

Edit the `Dockerfile` to look like this.

```
FROM khamoud/node-supervisor-mongo

COPY . /bundle
ADD supervisord.conf /etc/supervisor/conf.d/supervisord.conf

RUN apk add --no-cache make gcc g++ python && \
  cd /bundle/programs/server && \
  npm install && \
  npm uninstall -g npm && \
  apk del make gcc g++ python && \
  mkdir -p /var/log/supervisor && \
  mkdir -p /data/db

ENV PORT=80
EXPOSE 80
CMD ["/usr/bin/supervisord", "-n", "--configuration", "/etc/supervisor/conf.d/supervisord.conf"]
```

*The compressed size of this container is 124MB*

The `RUN` instruction does the following:

- adds all the necessary dependencies to compile the node modules
- `cd`'s into the bundle, and `npm install`'s everything
- after the `npm install` is complete, it removes all the compiler dependencies and `npm`
- makes two directories  

We do this all in one step to decrease the number of layers being used which will ultimately give us a smaller image.

### Step 5: Build and test the image

I'm going to build the image using my dockerhub username, but you should use your own.

`docker build -t khamoud/mongo-meteor .`

Next, run the image and see if everything worked according to plan.

`docker run -d -p 80:80 khamoud/mongo-meteor`

![Todos List with Meteor.js, mongo, and supervisord in one docker container](/supervisord-meteor-demo.png)

### That's it!

You can see the image running [here](https://todos.rompliapp.com).
