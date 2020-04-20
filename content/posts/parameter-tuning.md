---
title: "Parameter Tuning XGBoost, CatBoost, and LightGBM with Kubernetes"
date: 2020-04-20T11:11:49-04:00
draft: false
description: "parameter-tuning"
slug: "parameter-tuning-with-k8s" 
tags: []
categories: []
---

[Video Series Link](https://www.youtube.com/watch?v=6sZKiSbcwTc&list=PL63RY_Ityi0wwTaTvf_nFSn_z_LTmctpC)

The time is 2020, perhaps no other word can evoke quite a visceral reaction like "Kubernetes", [if](https://news.ycombinator.com/item?id=20915626) [uttered](https://news.ycombinator.com/item?id=18128456) [to](https://news.ycombinator.com/item?id=22491170) [a](https://news.ycombinator.com/item?id=18111665) [group](https://news.ycombinator.com/item?id=18953647) [of](https://news.ycombinator.com/item?id=19467067) [engineers](https://news.ycombinator.com/item?id=16285192). 

I certainly don't want to add to the noise. I can't see far into the [future](https://circleci.com/blog/its-the-future/) and predict if Kubernetes is ['it'](https://www.youtube.com/watch?v=LV0wTtiJygY). But from a data science perspective, Kubernetes solves a number of real-world thorny problems. The most important of which is that it enables DS folks to request resources(CPU, GPU, RAM, Storage) and deploy their applications themselves, as many and as often as they want, without requesting any help from the operations team. It is a well-known anecdotal fact that data scientists spend 80-90% of their time cleaning data, and 10-20% building models; a less known but equally true anecdote is that 80-90% of data science projects are deployed to PowerPoint slides or kept in Jupyter Notebooks. Data scientists often hope to hurled the Jupyter Notebook over an imaginary wall, to either the dev team or Ops team, for production and deployment. But maybe it's the separation of expertise/concern, maybe a catch-22, it almost never works.

Kubernetes is something that might solve this problem. It simplifies the resource request/deployment process to a point where any data scientist can learn the skills needed to take their model from conception, to experiment, and finally to production. It's always been possible, the bar is just lower with Kubernetes. The problem is that both Kubernetes and ML are relatively new topics, so tutorials and articles covering how to combine them are relatively hard to find.

I made this video [series](https://www.youtube.com/watch?v=6sZKiSbcwTc&list=PL63RY_Ityi0wwTaTvf_nFSn_z_LTmctpC) (and the [YT channel](https://www.youtube.com/channel/UC18byX5bLRU8zdADDzgJh8A)) in hope to bring out more coverage in this niche field of ML and DevOps. The topic I've chosen is a relatively well-known one for data scientists: parameter tuning. Most data scientist should be familiar with `Pool()` or `n_jobs=-1` at least in a local environment. This series hopefully shows how to replicate this process in a distributed environment. The game-plan is to first build a Docker container that accepts parameter grids in JSON as training parameters, then save the result to database. Next we generate a large number of such JSON messages, and stored them as jobs in a queue. We then connect our container to the queue; it executes each job and saves the result to the database. We then sort the result in the database to see which particular parameter did best. Since our container is managed by Kubernetes, we can scale the number of containers from 1 to hundreds, effectively giving us hundreds of CPUs to run the parameter tuning. 

The target audience for this video series is a data scientist that is somewhat familiar with Kubernetes concepts. I didn't want to do a [Kubernetes for beginner video](https://www.youtube.com/watch?v=r0uRLhrzbtU&list=PL2We04F3Y_43dAehLMT5GxJhtk3mJtkl5), since I think there are plenty of excellent [resources](https://www.amazon.com/gp/product/1617293725/) in this area, and I don't have much to add. What I wanted to do is to show one instance of how to combine ML and Kubernetes, even if it is for a unrealistic project. I wanted to show how easy it is to deploy something in Kubernetes. And hopefully you can walk away with some ideas to how to apply Kubernetes to your workflow.


I hope you find this video useful. Feel free to [reach out](https://yangyu.io/about/) via email or [twitter](https://twitter.com/Yangyu505).
