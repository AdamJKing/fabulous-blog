
> Prerequisites; knowing and using Docker is a must-have to get the most out of this technique. I'll be using ScalaTest, but there are wrappers for other libraries too.

The code for this blog can be found [here](https://github.com/AdamJKing/blackbox-testing-sample).

For the uninitiated, docker-compose is a tool that can build, deploy, and manage multiple containers on the same network. These configurations, known as compositions, remove a lot of the manual setup we often associate with running dependent infrastructure locally. A common use-case for these are providing local versions of dependencies for a faster, more realistic feedback loop when developing applications. Similarly, in “black-box” tests the aim is to validate the application's behaviour without changing how it works internally (think integration, or acceptance tests). Often you'll see code-bases setting up these dependencies in a way that results in messy or error-prone code. These are exactly the same problems that Docker, and Docker composition, help solve.

This is an example of how simple our application test can be.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FAdamJKing%2Fblackbox-testing-sample%2Fblob%2Feed9b2a6bef9fa2049c9be1908415ee43c3a9347%2Fintegration-tests%2Fsrc%2Ftest%2Fscala%2Fblackbox%2Ftesting%2Fsample%2FAppSpec.scala%23L19-L34&style=a11y-light&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

For this approach we'll be using [test-containers](https://github.com/testcontainers/testcontainers-scala) which provides a Docker context and native integrations for a variety of test frameworks. This framework is quite powerful by itself but one of the most useful features is the ability to run Docker compositions directly. 
