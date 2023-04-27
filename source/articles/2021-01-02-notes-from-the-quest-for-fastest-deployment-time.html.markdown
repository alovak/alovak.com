---
title: "Notes: The Quest for the Fastest Deployment Time by L Körbes"
date: 2021-01-02
tags: devops, engineering, operations
layout: post
public: true
---

What an awesome talk! I'm trying to ask myself why I'm so excited about it? What triggers my excitement? The answer is pretty simple. [L Körbes](https://twitter.com/ellenkorbes) in her talk "[The Quest for the Fastest Deployment Time](https://www.youtube.com/watch?v=WgliN_9j91g&list=PL2ntRZ1ySWBfUint2hCE1JRxRWChloasB&index=16)" (from [GopherCon 2020](https://www.youtube.com/playlist?list=PL2ntRZ1ySWBfUint2hCE1JRxRWChloasB)) shows how to decrease the time I (and many developers) waste by waiting... In other words, she makes our lives longer by giving us back lost time so we can think and do something important in our life. Thanks a lot! Also, this talk is a pleasure to watch. Great content, unusual production (switching cameras and perspectives), live coding, and jokes!

While watching the talk I remembered how I measured the time to deploy **Ruby** service to AWS ECS/Fargate (in production). It was for 2 minutes!
<center>
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Have measured how much time it takes from push to live: 120 seconds. It includes git push, build docker image, push to registry, update AWS ECS service (replace old containers with new ones). <a href="https://t.co/ALKe3DVobF">pic.twitter.com/ALKe3DVobF</a></p>&mdash; Pavel Gabriel (@alovak) <a href="https://twitter.com/alovak/status/977174782641950720?ref_src=twsrc%5Etfw">March 23, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 
</center>

Trying to be productive and effective, sometimes I struggle from not being able to see my results in production right after it's DONE*. Ok, in a few minutes. Usually, setting up a delivery pipeline is the first thing I implement after a walking skeleton is done. Having tons of tools and platforms these days lets you do this relatively easily.

About fast feedback, I heard first from Kent Beck. It became one of my personal engineering principles. It's not only about compilation time. It's about getting fast feedback from your tests (unit, integration, acceptance, etc.). It's about seen your code does the right (or wrong) thing for the customers. The whole Agile is based (at least it was so 10 years ago) on the feedback loop and its speed: direct conversations with customers, short iterations, spikes, TDD, BDD, CI, and CD.

While the talk is not directly about deploying to production, but about running your changes in the development cluster, there are still a lot of things you can apply to both (or your N environments). This talk has a lot of tricks for Golang developers, but developers using other languages will benefit as well.

Alright, now we can go straight to the notes and my key takeaways from the talk:

Three important rules to achieve the fastest deployment time:

1. don't rely on CI for development feedback (as an example, don't wait for your dev images to be built and pushed by CI to test them locally within your dev cluster)
2. use MDX (multiservice development experience) like [Tilt](https://github.com/tilt-dev/tilt), [Garden](https://github.com/garden-io/garden), [Skaffold](https://github.com/GoogleContainerTools/skaffold), [Telepresence](https://github.com/telepresenceio/telepresence/)
3. then use every hack and cheat to make it even faster

Specifics:

- use a tool to automate deployment into your development cluster (see point 2 above)
- track how much time it takes to complete each step of your deployment pipeline (get dependencies, compilation, docker image building, uploading, updating cluster config, etc.)
- cache your dependencies (using docker layer or vendoring them)
- use compiler cache (if you have a compiler :D)
- live update → do not rebuild/replace your docker container, just update the code and recompile inside container (using something like [entr](https://github.com/eradman/entr) to track file changes and run compiler when they change). Live compilation is not really good for production (because of security, build tooling inside your image, etc.)
- do not use the previous point if you don't have enough CPU (and memory) to compile fast :D Instead, compile locally an live update with binary
- when you send over the wire, reduce the size of the payload (your binary) → remove debugging data to make Go binaries smaller and/or
- use [UPX](https://github.com/upx/upx) (the Ultimate Packer for eXecutables) to compress your binary. This works for remote cluster. With compressing binary you lose on packing time which is important for your local cluster, but win on uploading time which is crucial for remote cluster.

[*]: follow your definition of DONE :)
