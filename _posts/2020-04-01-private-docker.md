---
layout: default
title:  Deploy a Private Docker Registry
date:   2020-03-01 22:47:53 -0500
permalink: /posts/private-docker
---


I was interested in setting up a private Docker registry for at least two reasons.

- I've accumulated, and subsequently lost track of, quite a few custom images. Maybe following a proper `build`, `tag`, `push/pull` flow workflow could end up saving me time in the long run.

- I also have considered creating a service that could evaluate models trained with different popular ML frameworks on a dataset unknown to the submitter. The first step in this process likely involves configuring a private registry to push Pytorch, OpenCV, Tensorflow, etc. containers.

This project never quite got off the ground. I was quite glad to complete the first stage and just have a personal registry running. A simplified version of what I'm running on a Pi in my closet is available [here](https://github.com/DMW2151/demo-docker-registry).

## Things I read to Make this Happen

- [Post](http://blog.johnray.io/nginx-reverse-proxy-for-your-docker-registry) on configuring nginx to reverse proxy localhost.
