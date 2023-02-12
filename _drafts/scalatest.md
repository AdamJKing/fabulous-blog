---
layout: post
title: Expressive Assertions With ScalaTest
readtime: true
---

> This post assumes a basic familiarity with the [ScalaTest](https://www.scalatest.org/) testing framework as well as the Scala programming language.

Many developers will end up working on code that has already been developed and is, for the most part, fully tested. In these situations we might just come across code that looks like this:

<!-- replace with scastie example -->
```scala
"The CoolDudeSimulator" should "parse the upstream Nanobots correctly" in {
    val  simulator = new CoolDudeSimulator()

    simulator.processNanobots(500).isRight shouldBe true
}
```

Seems simple enough right? We run the test suite and the test passes - we don't need to think too much about it. As ever though the cruel spectre of change appears in our sprint backlog and it's time to make a change to our code's behaviour. We update the `CoolDudeSimulator` to process the `Nanobots` using a smarter algorithm but when it comes time to run our tests:

```
placeholder, true was not equal to false
```

Hmm, how could that be? It seems our implemetation wasn't up-to-scratch and worse yet, we have no idea why! We notice that another test is comparing the result directly to a known result and decide to copy it.

```
placeholder, X shouldBe Y resulting in a large comparison
```

Things have not really improved. Now instead of a comparison with too little information we have a comparison with too _much_ information.  Let's take a look at our trait definition to see if we can break down the problem.

```scala
trait CoolDudeSimulator {
  def processNanobots(nanobotQueueId: Int): Either[ProcessError, Option[List[NanobotSummary]]]
}
```

That's a gnarly return type. I've intentionally complicated it more than neccessary but it's certainly possible for type signatures like this to crop up. Let me provide an explanation:

```scala
Either[ProcessError,        // Our processor could fail for technical reasons.
    Option[                 // The queue isn't validated until processing and it could be missing.
        List[               // We get the processing results for multiple entries at once.
            NanobotSummary  // The interesting information about our Nanobots.
        ]
    ]
]
```

Hopefully I've convinced you this could really happen, or maybe you've seen something similar in the wild. Normally we would like to refactor interfaces like these, perhaps validating our queue IDs beforehand to remove the possibility of the queue being optional. Unfortunately we don't always have the luxury of a big refactor and sometimes code like this is untested.

<!--

Flaws with  basic testing approaches

Useful techniques for testing
- nested datatypes
- flexible requirements
- code as documentation

-->

<!-- <script src="https://scastie.scala-lang.org/xbrvky6fTjysG32zK6kzRQ.js"></script> -->