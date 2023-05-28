---
created: 2023-05-11T11:10:46+01:00
modified: 2023-05-28T21:32:45+01:00
---

# Black box testing with docker composition

Test containers
Docker deployments
Local development transition into testing
Tests part of the code that doesn't always get tested

> Prerequisites; knowing and using Docker is a must-have to get the most out of this technique. I'll be using scalatest but there are wrappers for other libraries too.


"Black box" testing, sometimes known as integration or acceptance tests, are a type of automated test which operate outside or without the internal context of the application. In the case of web servers this usually involves spawning an instance of the application to which we submit real HTTP requests. Managing the execution of these fixtures during test can be particularly cumbersome; requiring process management, mock dependencies, and local environment variables.

It's not unusual to see this kind of setup relegated to custom test runners or the build tool we're using. In modern cloud based services it's very common to *Dockerise" our applications which allows us to define an environment for our app without concern for where it is actually running. Docker provides

-too high level, bring back to target audience
