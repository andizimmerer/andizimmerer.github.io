---
title: "Mission Impossible: Integrating Xtext, QVT-o and Acceleo into your Gradle Build Pipeline"
date: 2019-08-04
summary: "Usually, tools from the EMF ecosystem are tighly coupled to Eclipse. However, we managed to integrate Xtext, QVT-o and Acceleo into the Gradle build pipeline in one of our projects, making it possible to use our product without installing Eclipse."
draft: true
---


# Introduction and Overview

# Prerequisits for this Tutorial
This tutorial assumes that you already know the basics of [EMF][10], [Xtext][16], [QVT][13] and [Acceleo][15].
It's also good if you know some basic stuff about Ecore (see [its documentation][17]) and metamodels (the backbone of EMF).
See the [EMF tutorial][18] to get you started with the basic concepts.

# Project Background
During the development of "[Plantestic][1]" we were facing the following problem:
We wanted to make use of the large [EMF ecosystem][11] but we didn't want to force anybody who was using "Plantestic" into using Eclipse.

A short overview on what "Plantestic" does to give you are rough idea if this is the right blog post for you:

Plantestic allows you to convert UML Sequence Diagrams into tests for your REST API.
Most of the time when companies use contractors for their projects, they also include UML sequence diagrams for specifying the project's API.
When the project is done, they quickly want to test the result against the specification but this costs additional time and money to do so.
Therefore, it is desirable to generate API tests based on the specification.
We actually did this project for a large company who approached us with exactly this problem and they were very happy with the final result.
You can find the project [on GitHub][1].

Plantestic is doing the following steps to generate test cases:

 1. Parse a UML sequence diagram written with [PlantUML][12]. 
    The advantage of using PlantUML is that it is very easy for the user to quickly create UML diagrams.
    The downside is that the resulting diagrams hardly follow the official UML standard by means of semantics and they also only "look like" UML diagrams but they lack a lot of information that is usually provided when using real UML.
    But customer first means: don't force them into using something they don't need!  
    We use [Xtext][16] to create a parser for "our custom DSL" (=PlantUML syntax) because this gives us an output in the ECore format which allows us to easily integrate it with other EMF tools.
 1. Because the output of Xtext is highly specific to PlantUML, we want to transform the diagram in a very generic intermediate representation.
    Think "getting rid of arrows and colors and use only their semantic information like 'what is a request'".
    We use again a tool from the EMF ecosystem: [QVT-operational][13].
    Another contentor of QVT is ATL.
 2. Now we want to transform the very generic intermediate representation into a very specific output representation.
    Think "From semantically rich meta-objects to concrete Java classes".
    We again use QVT for this.
 3. Now generate actual source code for the tests from the model.
    Because we did step (3) this is very straight forward.
    We use [Acceleo][15] from the EMF world to do this.

All in all we use the tools Xtext, QVT-o and Acceleo from the EMF ecosystem to generate source code from a given UML sequence diagram.
However, all these tools are heavily integrated into Eclipse (I mean, there is a reason why EMF stands for "Eclipse Modelling Framework").
Nevertheless, we wanted to develop a standalone tool which was easy to use for our end-users without forcing them to install Eclipse.

It took us some sleepless nights and we dug deep through undocumented source code until we figured out every single step needed to make all this work.
In the end, the solution does actually not require that many lines of code.
By publishing this post I hope to save you from some days of headage.
Enjoy the journey!

## Why EMF?
[TODO: should I even include this section?]

## Current Workflow with EMF and Xtext, QVT-o and Acceleo

# Reasons why you want to do This
Maybe the big elephant in the room is:
"Why should you even do this?
For years it was good enough to use Eclipse for this.
It gives you a lot of features and does everything for you.
Why should we stop using Eclipse for doing EMF projects?" 

Well, in our specific case the answer is quite easy:
We want to develop a standalone tool because our user is not interested in the technology behind it.
However, even if this is not a compelling reason for you, there are several others:

 - **Programmatically Invoke the Pipeline with full Control**:
   You can run the transformation pipeline from within your existing code and you have full control over it.
 - **Automated Tests**:
   You can easily write tests (e.g. with JUnit) to check your transformations.
   Specify an input, do the transformation step, check the output.
   You should always write tests for your code ;-)
 - **CI-Pipeline**:
   When following the two points above, you have a nice integration into your build pipeline.
   `gradle check` now builds your parser, checks the parser, checks the transformations and checks the output model.
   Nothing else is required.
 - **End-To-End Tests**:
   We were able to write some very nice End-To-End tests for "Plantestic":
   Take a `.puml` file, run the transformation pipeline on it, _dynamically compile the resulting source code_ during the test, spin up a configured mock server and run the generted test case against the mock server!
   Gosh, this is really what we needed during the project to check that everyhting works as expected!


# Limitations (Or: Reasons why you don't want to do This)
The benefits of integrating Xtext, QVT-o and Acceleo directly into your build pipeline also comes with a few drawbacks:

 - **No Syntax Highlighting**:
   This is only a minor one, but when working with Xtext grammar (`.xtext`), QVT-o transformations (`.qvto`) or Acceleo templates (`.mtl`) you have no syntax highlighting.
   However, you can circumwent this problem by using Eclipse with the corresponding EMF plugins for editing.
 - **No Auto-Completion**:
   Because we gave up the dependency on Eclipse we also don't get the benefit of automagical auto-completion.
   We could not find a way of enabling auto-completion when editing the transformation files with Eclipse; however it should be possible in theory.
 - **No fat-JAR (yet)**:
   Wait, what? Did I really just say that?
   Yes, but hold back! This is just an issue we encountered with Acceleo.
   Xtext and QVT work just fine when bundling them into a fat jar.
   Why does Acceleo prevent us building a fat JAR?
   Well, when executing an Acceleo transformation, there is still one intermediate step involved, namely compiling the `.mtl` file to an `.emtl` file.
   Because we are doing this at runtime, it is not distributed within the JAR archive.
   But behold, there are silver lining on the horizon:
   The nicest solution would be to write a gradle task that compiles the `.mtl` file to an `.emtl` file at build time and packages it within the JAR archive.


# How we achieved our Goals

Below you'll find an excerpt of our `build.gradle` with a list of all required dependencies for QVT and Acceleo.
Most of them are directly from the Eclipse repository.
To include the Eclipse repository you need to add `maven { url "https://repo.eclipse.org/content/groups/releases/" }` under the `repositories` section in your `build.gradle`.
You also need to include `maven { url "https://dist.wso2.org/maven2/" }` for the other dependencies.

<script src="https://gist.github.com/Jibbow/ae7bcae6b0d74e119d718a3080086e65.js"></script>

## Xtext as a Parser Generator and Model-Generator

## QVT-o for Model-To-Model Transformations

## Acceleo for Model-To-Text Transformations

# Conclusion
Yes, it is possible to have a nice build pipeline with EMF-Tools!
And in hinsight it is not even that complicated after you figured out how to do it.
The downside is that there is almost no documentation available on doing this and most resources assume you have everything running inside Eclipse.
Nevertheless, we created a quite robust way of integrating these three tools in our pipeline with full control over the workflow, which is quite nice.

If you intend to use one of the frameworks covered in this post, I strongly recommend you to follow our approach as this gives you the advantages of automated tests, an good integration into you build pipeline and an easy-to-understand workflow for new developers without the need of using Eclipse.

The downside of this approach is that you don't get the benefits of a full IDE integration: no syntax highlighting and no auto-completion while working on QVT-o transformations or Acceleo generations.


[1]: https://github.com/FionaGuerin/plantestic "Plantestic"
[2]: https://github.com/FionaGuerin/plantestic/blob/master/core/build.gradle#L74 "Making QVT work without Eclipse"
[3]: https://github.com/FionaGuerin/plantestic/blob/master/core/build.gradle#L91 "Making Acceleo work without Eclipse"
[4]: https://github.com/FionaGuerin/plantestic/blob/master/core/build.gradle#L52 "Additional Maven Repositories"
[5]: https://github.com/FionaGuerin/plantestic/tree/master/plantuml "The Xtext sub-project"
[6]: https://github.com/FionaGuerin/plantestic/blob/master/plantuml/src/main/java/plantuml/GeneratePumlLanguage.mwe2#L14 "Missing line in generated Xtext project"
[7]: https://github.com/FionaGuerin/plantestic/blob/master/build.gradle "The root build.gradle"

[8]: http://www.davehofmann.de/different-ways-of-parsing-with-xtext/ "Standalone Xtext"
[9]: https://wiki.eclipse.org/QVTOML/Examples/InvokeInJava "Invoke QVT-o from Java"

[10]: https://www.eclipse.org/modeling/emf/ "EMF Overview"
[11]: https://www.eclipse.org/modeling/transformation.php "EMF M2M Transformation Technologies"
[13]: https://projects.eclipse.org/projects/modeling.mmt.qvt-oml "QVT-operational"
[14]: https://projects.eclipse.org/projects/modeling.mmt.atl "ATL"
[15]: https://www.eclipse.org/acceleo/ "Acceleo Model-To-Text"
[16]: https://www.eclipse.org/Xtext/ "Xtext Parser Generator"
[17]: https://download.eclipse.org/modeling/emf/emf/javadoc/2.9.0/org/eclipse/emf/ecore/package-summary.html#details "Ecore Documentation"
[18]: https://eclipsesource.com/blogs/tutorials/emf-tutorial/ "EMF Tutorial"

[12]: http://plantuml.com/ "PlantUML"
