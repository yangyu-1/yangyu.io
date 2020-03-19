---
title: "Tuning K8s 0"
date: 2020-02-23T19:42:17-05:00
draft: true
---

Many data scientist are starting to ask `What is Kubernetes?`, `Why is it useful?`, and `How can it help with ML workloads?`. Despite the strong interest, however, Kubernetes is still a relatively new technology. There aren't even many useful tutorials on Kubernetes itself, let along tutorials on Kubernetes and ML.

To better understand how Kubernetes can help data scientists, I'm going to take a well understood ML problem, namely parameter tuning, and use Kubernetes to build and deploy a pipeline for parameter tuning across 3 different packages: XGboost, LightGBM, and Catboost. If you ever wonder how to run parameter tuning on 500 cpu cores instead of just your laptop, this video will show you how to do that.

The video will utilize Google Cloud Kubernetes Service, call GKE. As of the time of this video, Feb, 2020, Google Cloud Platform is still the easiest platform to get some free credit. Making this video I've spend about $10 out of the $300 credit provided at signup.

I often find that having a high level overview is a useful starting point before diving into the details, so let's start with that. 

We begin with a parameter grid. Then in a Jupyter notebook, we'll break the grid into individual parameters, encode them to json, and send them to our workers. Wokers will load the json message, decode it back to parameters, feed the parameters into a model and start the training. As each training task completes, we'll save it to a database.

Knowingly or not, this is a process many data scientist use when doing parameter tuning on their laptops. Instead of workers, we use Scikit-learn's `n_jobs = -1` option or Python's Pool() class to distribute tasks among Python subprocesses. Because of Python's Global Interpreter Lock, however, this process is often restricted to how many cores are available on a single machine.

With Kubernetes, it becomes possible to distribute the tasks among 10s, 100s, even thousands of CPU cores. But the process is significantly more difficult, and often the code will have to be re-written from the ground up. There is simply no way around that.

Before we get started, a disclaimer: This video series is intended to be a demo, not a full tutorial on Kubernetes. If you would like to learn more, check out the blog post in the description where I recommend some useful learning resources. None of the links are affiliate links, I'm not making any money off of them; they're just resources that helped me learned Kubernetes. The blog-post also contains video transcripts, github links, and more. If you're never used Kubernetes or similar devopsy tools before, this video might be somewhat confusing to you. I would encourage you to watch the first video, then seek out some resources to learn about Kubernetes, and comeback to watch it again. You'll see that Kubernetes is ultimately not that complicated. 

Furthermore, in reality, there are much simpler implementations to do parameter tuning than building out a Kubernetes pipeline. This is a demo to showcase Kubernetes. In the blog post I talk about some realistic options if your goal is to achieve the best results with your model.

In total, this will be a 3 part series. In the remainder of this video, we'll dive into more details on how the training pipeline is setup, and finish with a demo of the final, working product. 

In the next video, we'll start from scratch and develop a working local version of the pipeline, before deploying most of the parts to GKE

Finally, we'll deploy our database to using Kubernetes and complete the project. We deploy the database last because that is the hardest part to deploy.

Let's dive into the details of pipeline. We start with a jupyter notebook containing our parameter grid. We'll break the parameter grid into individual parameters, and encode each parameters into JSON messages. Next, we send our JSON messages to a message queue, we'll use RabbitMQ in this case. On the other side of the message queue we'll have a number of workers, listening to the queue. Once a message is within the queue, our workers will pull the message out, decode the message and feed the parameter into a model. If there are a lot of messages, we can simply scale up the number of workers to handle the load. As each worker finishes training, we'll save the data to a mongodb database.

Of the 4 components listed here, the Jupyter notebook, the message queue, the workers, and the mongodb database, the message queue and the database are readily available. We'll simply use RabbitMQ and MongoDB's official docker image on DockerHub. The Jupyter Notebook will always be local and lives completely outside of Kubernetes. That leaves us with the worker, which we'll write the code, build the DockerImage, push it to DockerHub, and deploy using Kubernetes. 

In this tutorial, we'll use Kubernetes to deploy and manage message queue, the worker nodes, and the database. We could abstract away the message queue and database by using existing cloud solutions, such as GCP's Pubsub, or AWS's DynamoDB, but this is a demo after all, so we'll build them ourselves.

So let's get started:
1. We'll go to the Github link in the blog post and clone the repo. What you see on your screen might be slightly different than what I'm showing here, since I'm still finishing up the demo.
1. Once the repo is cloned. CD into the "final" folder. 
1. Create a virtual environment, install everything in the requirements.txt, start the jupyter notebook
1. The first piece we'll deploy is RabbitMQ. Let's take a look at the YAML file. Couple things of note.
    - First of all, notice we have two components inside this YAML. One is for RabbitMQ, which we're using the deployment resource to manage. The other component is a service resource. Without service, there is no way for other resources inside our Kubernetes cluster to communicate with our message queue. As you'll see later in the worker's yaml file, workers use "rabbitmq-service" to connect to the message queue.
    - Notice the image we're using. This is pulling the official rabbitmq image from Dockerhub. It's an official Docker image from the RabbitMQ team. It's not something we created.
    - We open two ports, 5672 is the port for push and pulling messages, while the 15672 is a management port with an UserInterface that we can see in the browser.
    - Notice the replicas is set to 1. That means we're only running a single copy of the message queue. 
    - The last thing to note is the nodeSelector option. In this video series I have two pools of virtual machines. The default-pool are normal GCP VMs, while the pool-1 is use exclusively by the worker nodes. The reason for this is to cut down cost by using spot instance for workers, also called preemptible machines on GCP, meaning they can be shutdown at anytime but is much cheaper than normal VMs. If the VMs hosting our workers is shutdown, it'll have no effect on our pipeline. On the other hand, we can't afford to have our message queue, or database shutdown unexpectedly. 
1. Let's first run `kubectl get pod` and `kubectl get deployment` to make sure nothing is running inside our Kubernetes cluster. 
1. Then let's increase our default-pool VM instance to 2.
1. Next we'll deploy the rabbitMQ deployment and service by using `kubectl apply -f rabbitMaster.yaml`
1. Now let's wait for our deployment to be created.
1. Once our deployment is created, we need a way to connect to our rabbitMQ server. By default, no external traffic is allowed to connect to anything running inside of our cluster. In a production environment, we would create an ingress, nodePort, or load-balancer to expose our rabbitmq deployment to external traffic. But for this demo, we're simply going to use port-forwarding to access our rabbitmq deployment. 
1. Run `kubectl get po` to get the pod name for our rabbitmq deployment.
1. Use the command `kubectl port-forward {name of pod} 5672:5672 15672:15672` to forward the traffic to localhost.
1. Once that's done, we can now use our browser to navigate to localhost:15672 to see the RabbitMQ management UI.
1. I feel the need to take a moment and see what we've done so far. You now have a rabbitMQ service running in the cloud! That is not trivial! A few years ago, you would've have to rented a VM, SSH into it, download and install rabbitMQ, harden the VM, create a load-balancer for it, and then you can finally connect to it. Now with Kubernetes, the process is much simpler, and you get features such as self-healing, resource limitation, and declarative syntax for free.
1. Now let's take a look at our worker file. Worker file is slightly more complicated. There are two parts. The first part is the code, which is stored in the worker/ folder. The second part is the worker.yaml file, which, similar to our rabbitMaster.yaml file instructs kubernetes how to deploy the workers. Here are couple things to note:
    - Inside the worker.py file in the worker folder, we see that it reference a environmental variable call rabbitService. You might be wondering where that's from. After all, we named our rabbitMQ service `rabbitmq-service`. The way we pass environmental variable to our pods is through a Kubernetes resource call configMap, which can be found in woker.yaml.  It's similar to a key value store. The line we're looking at is the `rabbitService: "rabbitmq-service"`. Similarly, we also pass in the mongodb service address via environmental variable. 
    - The next thing we want to take a look is the callback function. This functioned is called every time our worker reads a message off the queue. The body variable contains the message, which is a json string. The json data is passed to a function call trainer(). This function is defined in the trainer.py file. In short, it trains the model using parameters from the JSON string passed in. Once the training is done, the trainer calls a save_to_mongo function to save the results to our mongodb database. 
    - You can run `docker build -t {yourDockerHubName}/{yourDockerImageName} .` inside the worker folder to build the image. Then run `docker push {yourDockerHubName}/{yourDockerImageName}` to push the data to Dockerhub. For me, the image is `yuy910616/rabbit-worker`, which is referenced in the worker.yaml file.
    - There are a lot more details in the workers.yaml file. We'll go into more details in the next video.
1. We'll now increase our pool-1 instance to 1. We'll only use 2/64 cores on the machine for now.
1. Next thing we need to do is run `kubectl apply -f worker.yaml` to deploy our workers.
1. Run `kubectl get pod` and `kubectl get deploy` to see our new deployment
1. We can see in our RabbitMQ UI that we should now have 2 workers connected to our queue.
1. We can run `kubectl describe pods {nameOfPod}` to see if the pod is ready. Or use `kubectl logs -f {nameOfPod}` to stream the logs
1. We're 3/4th of the way there. Now the last thing to do is deploy our database. Storage and managing state has always been a challenging part of Kubernetes. Luckily, our setup for this demo is quite simple: we're effectively running a single server database. Some of the elder folks watching this video might even remember running such configuration in production. There is no backup, no hot-swaps, no replication, no failover, but for our demo, it's more than enough. 
1. Let's take a look at our mongodb.yaml file:
    - You can see that even our simple setup requires us to use 4 resources: a PersistentVolume, a PersistentVolumeClaim, a ReplicaSet, and a service.
    - We'll go into details of why we need each resource in the third video.
1. For now let's run `kubectl deploy mongodb.yaml` to deploy our database.
1. To check if whether or not our database is online, we'll do two things:
    - First let's run `kubectl get pods` to see if the pod is ready
    - Next, we'll run another `kubectl port-forward {name of Mongo DB pod} 27017`, and connect to our pod using a mongoDB GUI tool, call Compass.
1. With all of 4 of our components deployed, let's now return to our Jupyter Notebook.
1. Inside the notebook, I have a function call split_params that splits the parameter grid into n chunks.
1. For example in this example, I'm splitting the parameter grid, which contains a total of 26160 different parameters, into 5000 different chunks, and each chuck contains 6 different parameters. This means we have 5000 jobs will be send to our workers.
1. As the jobs finish, we should start seeing the results roll into our MongoDB database.
1. Now for the final magical moment, how do we scale? It's a simple command `kubectl scale --replicas=40 deployment/rabbitmq-worker-deployment`.
1. If we need to scale up some more, we can increase the number of nodes inside pool-1, and keep going up from there.
1. This concludes our demo portion. 