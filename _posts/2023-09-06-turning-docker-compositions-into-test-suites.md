---
title: Turning Docker Compositions into Test Suites
layout: post
---

> Prerequisites; knowing and using Docker is a must-have to get the most out of this technique. I'll be using ScalaTest,
> but there are wrappers for other libraries too.

The code for this blog can be found [here](https://github.com/AdamJKing/blackbox-testing-sample).

For the uninitiated, docker-compose is a tool that can build, deploy, and manage multiple containers on the same
network. These configurations, known as compositions, remove a lot of the manual setup we often associate with running
infrastructure locally. A common use-case is providing local versions of dependencies for a faster,
more realistic feedback loop when developing applications.

Similarly, in “black-box” tests (think integration, or acceptance tests) the aim is to validate the application's
behaviour without changing how it works internally. Often you'll see code-bases setting up these dependencies in a way
that results in messy or error-prone code. These are exactly the same problems that Docker, and Docker composition, help
solve.

This is an example of how simple our application test can be.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2Feed9b2a6bef9fa2049c9be1908415ee43c3a9347%2Fintegration-tests%2Fsrc%2Ftest%2Fscala%2Fblackbox%2Ftesting%2Fsample%2FAppSpec.scala%23L19-L34&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

### Test Containers

For this approach we'll be using [test-containers](https://github.com/testcontainers/testcontainers-scala) which
provides a Docker context and native integrations for a variety of test frameworks. This framework is quite powerful by
itself but one of the most useful features is the ability to run Docker compositions directly. Let's start by defining a
simple composition file.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2Feed9b2a6bef9fa2049c9be1908415ee43c3a9347%2Fdocker-compose.yaml&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

I've kept it simple by only including our application, but from this point on I can extend it with whatever setup we
need without having to change our tests at all. In order to start using this in our tests we'll need to do three things.

1. Build and publish our application image locally.
2. Integrate our composition setup into our test-suite.
3. Create a client which is capable of calling our application.

### 1. Build and publish our application image locally.

Docker relies on our application being available as an image before we can use it in a suite. Unfortunately
docker-compose does not have a concept of running external build tasks to acquire an image. That being said most build
tools will allow us to build an image before running our test suite. With SBT and the native-packager plugin it's as
simple as adding our Docker build stage as a prerequisite for our integration tests.

> An SBT quirk means we need to specify our Docker build step for every test task.

```
    Test / test := (Test / test).dependsOn(server / Docker / publishLocal).value,
    Test / testOnly := (Test / testOnly).dependsOn(server / Docker / publishLocal).evaluated,
    Test / testQuick := (Test / testQuick).dependsOn(server / Docker / publishLocal).evaluated
```

### 2. Integrate our composition setup into our test-suite.

The next step is to wire in the details of our image into our tests. The test-containers library has integrations for a
variety of test frameworks, which makes this easier, but we'll still need to do a little wiring. The
docker-compose module needs to know the location of our compose-file. It might be tempting to hard-code this, but that
will affect the portability of your tests. Instead, it's better to pass this in as a property and use SBT to figure out
where the compose file is. This has the added benefit of being able to test against multiple compose-files
which is useful if we wanted to run our tests against different environments.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2F388fe4e99956353c83e99f38f60a1c90f903c3fc%2Fbuild.sbt%23L39&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

The caveat here is that if you prefer to use an IDE to run test-suites you may need to update the respective run config
before this will work. In IntelliJ, you just need to edit the run configuration which you can share by saving in your
project repo.

![IntelliJ Run Configuration](/assets/images/IntelliJRunConfig.png)

The Docker context is not available immediately so any information we need has to be lazy, or use the test-containers
helper. This becomes relevant when we want to distribute these details in our tests.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2Fmain%2Fintegration-tests%2Fsrc%2Ftest%2Fscala%2Fblackbox%2Ftesting%2Fsample%2FAppFixture.scala%23L34-L36&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

The last bit of wiring we need to do is instructing test-containers on how to use our composition.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2Fabed609fdbe0c4a17dfedebf7005bbc89eead893%2Fintegration-tests%2Fsrc%2Ftest%2Fscala%2Fblackbox%2Ftesting%2Fsample%2FAppFixture.scala%23L39-L47&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

There's two key extra bits of information here. The first is that we tell test-containers at which point it should
consider the composition ready for testing. In this case I have added a wait condition for the service's health-check.
There are many approaches you can take here and if your setup is more complex it's worth checking out
the [test-containers documentation](https://java.testcontainers.org/features/startup_and_waits/#:~:text=not%20a%20daemon.-,Wait%20Strategies,container%20is%20ready%20for%20use).

Secondly we specify which container's logs we would like to follow. I have kept it simple by only following the main
container as this can become overwhelmingly quickly with multiple containers. It is possible to implement more
complicated logging solutions but if your containers are logging too much, it's best to try to configure logging
through the application itself.

### 3. Create a client which is capable of calling our application.

The final step is to create our HTTP client to call our service. Anything the Docker containers expose is available to
us and, if needed, we could connect to our dependencies directly.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2Fabed609fdbe0c4a17dfedebf7005bbc89eead893%2Fintegration-tests%2Fsrc%2Ftest%2Fscala%2Fblackbox%2Ftesting%2Fsample%2FAppClient.scala%23L12-L24&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

How you approach this will vary based on what you're testing and whether you'd prefer to distribute client libraries
yourself.

## Summary

With our test fixture code in place we can now start writing code without too much concern about the state of our
containers.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2Feed9b2a6bef9fa2049c9be1908415ee43c3a9347%2Fintegration-tests%2Fsrc%2Ftest%2Fscala%2Fblackbox%2Ftesting%2Fsample%2FAppSpec.scala%23L19-L34&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Now if we encounter a test failure we can launch our docker-composition to debug our test case. This helps us switch to
a more iterative feedback loop until we are satisfied with our application and test code.


