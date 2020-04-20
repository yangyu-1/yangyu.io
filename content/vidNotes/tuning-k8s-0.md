---
title: "Tuning K8s 0"
date: 2020-02-23T19:42:17-05:00
draft: true
---
Welcome to the first video of MLOps
For better or worse, Kubernetes is going to become an important part of infrastructure going forward.

There are many tutorials out there on how to use kubernetes. I'll take a slightly different approach, I'm going to use Kubernetes to build a pipeline for parameter tuning. 

We'll split a large parameter grid into smaller tasks, 

send them to a message queue. 

On the other side of the queue, we'll start some workers to work on these tasks, and save the results to a mongodb database. In essence, we're doing parameter tuning but with hundreds of CPUs cores distributed on a cluster of VMs instead of just one local machine.

During the process, we'll use Kubernetes to deploy the message queue,the workers, and the mongodb database. You'll get to see how Kubernetes can help you quickly and easily deploy and manage these components, 

as well as scaling out workers. But keep in mind, this tutorial is intended to be a showcase of Kubernetes, so it moves fast. It certainly would not teach you Kubernetes from the ground up; if you're interested in that, I shared a few resources I like in the blog post

The video will utilize Google Cloud Kubernetes Service, call GKE. The only reason why I picked them is because they're the most generous with sign up credits. 

For some reason I decided to do parameter tuning across XGBoost, Catboost, and LightGBM. Looking back this was probably too ambitious for this small project. If you're wondering if the material in these videos are production ready----they're not. In fact, writing this much infrastructure code just for parameter tuning is kinda like building your own PC from scratch: sure it's part utility, but mostly you're doing it for fun or to learn a thing or two. 

In total, this will be a 3 part series. In the remainder of this video, we'll dive into more details on how the training pipeline is setup, and finish with a demo of the final, working product. 
In the next video, we'll go over the rabbitMQ and worker yaml files. I'll explain in detail what each yaml file is doing.
Finally, we'll go over the mongoDB's yaml file and components. We'll discuss the database last because that's the hardest part.

I will now give a quick overview of the pipeline before jumping into the demo. 
It's a simple producer - consumer architecture. Essentially we produce a list of training tasks, then our consumer takes each task and process them in parallel. 

The tasks are the parameters from the parameter grid. We'll use some code in a jupyter notebook to break them up into json messages.

We'll send them to rabbitmq and our workers will connect to the rabbitMQ service process the messages.

Kubernetes will give us an simple API to increase the number of workers.

As each task completes, we'll save the results to our mongoDB instance

The workers is a custom docker image I wrote. For both rabbitMQ, and mongoDB, we're using their official image from dockerHub

Simple enough, let's take a look at how the completed product looks:
1. We'll go to the Github link in the blog post and clone the repo. What you see on your screen might be slightly different than what I'm showing here, since I'm still finishing up the demo. We'll click on the clone or download button, and copy and paste the link into our terminal
1. We'll do a git clone command follow by the link we copied. Once the repo is cloned. CD into the "final" folder. We'll do a quick ls command to make sure we have the right files.
1. Create a virtual environment, update pip, and install everything in the requirements.txt, which could take a minute. Once done, start the jupyter notebook.
1. We'll start a new terminal, because we need the Jupyter notebook to run in the background. We'll cd back into the folder, and open a code editor.
1. That completes the setup stage.

1. The first thing we need to deploy is our rabbitMQ message queue. The deployment configuration is specified in the rabbitMaster.yaml file. We'll go into more details about this file in the next video. For now, let me point out two things:
    1. First - Notice the image tag. This tells Kubernetes what Docker Image to pull from Docker hub.
    1. Second - Notice we open two ports: 5672, and 15672. 15672 is a management port, where we can see a graphical user interface for rabbitMQ. 
1. We'll deploy the rabbitMQ deployment and service by using the `kubectl apply -f rabbitMaster.yaml`.
1. We can check our deploymen  t by issuing `kubectl get deployment` or the short form `kubectl get deploy` in the command line. This command shows the status of our deployment.
1. We can check on the Pods created by the deployment. Pods are a lower level resource that are often managed by the deployment resource. If we have a pod that is not managed by the deployment, then if we accidentally kill it, or if the host machine dies, the Pod goes with it. But if the pod is managed by the deployment, the deployment will make sure there is always x number copies running, specified by the replicas field yaml file. If the host machine dies, the deployment will self-heal and schedule the pod in a different machine. We can check on the pod using `kubectl get pods` or `kubectl get po`
1. The next command we'll use is `kubectl describe ` follow by the the resource name, in our case, pod. or po. and follow by the pods name. This gets us more info on the individual pod. If something is stuck in pending, describe will show the error message of why it is stuck.
1. So how do we access our deployment? by default, no external traffic is allowed to connect to anything running inside of our cluster. In a production environment, we would  expose the rabbitmq server as an API, open to anyone who has the right credential. But for this demo, we'll use something simpler. Kubernetes comes with port forwarding, which connects to individual pods by port forwarding local ports. This is a great debugging tool, or if you just want to get something up and running really quickly.
1. Run `kubectl get po` to get the pod's name for our rabbitmq deployment.
1. Use the command `kubectl port-forward {name of pod} 5672:5672 15672:15672` to forward the traffic to localhost.
1. Once that's done, we can now use our browser to navigate to localhost:15672 to see the RabbitMQ management UI. And login using username guest and password guest. We can see that our node is the name of our pod, and connection is current zero, because we don't have any workers
1. So just like that, our rabbitmq server is up and ready.

1. Next thing we need to do is run `kubectl apply -f worker.yaml` to deploy our workers.
1. Let's take a look at our worker.yaml file. 
    1. First - Notice the image tag is now pointing to an image under user: yuy910616. That's me, and the rabbit-worker image is build using the source code in the worker folder.
    1. Second - For the workers we specified CPU and memory requests. Each worker is limited to 1 cpu and 750mb of memory.
    1. Lastly - notice we're creating 3 replicas, whereas for rabbitMQ we only had one. This means we'll have 3 different workers.
1. Run `kubectl get deploy` to see our new deployment. and then run `kubectl get po` to check if the new pods are running
1. We can see in our RabbitMQ UI that we should now have 3 workers connected to our queue.
1. We'll type in `kubectl logs -f {nameOfPod}` into the shell to stream the output to our terminal. This will output any print or log statements from our python code. We can see that the first print message is "[*] Waiting for messages. To exit press CTRL+C" and this comes from our source code, which is stored in the worker folder, in file worker.py

1. Finally, for the database. Run `kubectl apply -f mongodb.yaml` to deploy our database. The yaml file for deploying database is a lot more complicated. We'll go over it in the last video.
1. We can use `kubectl get rs` to check on our replicaset's status. We have an replicaSet this time instead of an deployment. An deployment would also work in this case.
1. Once we see the mongodb deployment is ready, we'll do another port-forwarding, this time using port 27017. We can then connect to it by opening Compass, and connect to mongodb://localhost:27017/. 
 
1. With all of 4 of our components deployed, let's now return to our Jupyter Notebook.
1. Inside the notebook, we'll execute the first cell, which just imports the necessary packages.
1. The second cell contains a list with 3 dictionaries, each is a parameter grid for a specific package. We also have a split params functions, that takes the parameter grids, and split them in n jobs.
1. Once we execute the second cell, we should see an output telling us the total number of parameters, the number of parameters inside each job, and the number of jobs. For this demo, we have about 26k parameters to search through, they're divided into 5000 jobs, with each job containing 6 parameters. 
1. We then use a package name Pika to send the job to our rabbitmq Deployment.
1. Switch back to our management UI, we see that now we have 5000 jobs waiting to be completed, and we have 3 workers working hard on them.
1. As the jobs finish, we can switch back compass, and we should start seeing the results roll into our MongoDB database.
1. Now for the final magical moment, what if we want to increase the number of workers? It's a simple command `kubectl scale --replicas=40 deployment/rabbitmq-worker-deployment`.
1. we can use the commands like kubectl get deploy and kubectl get po to see if the command worked. As you can see from my screen, the desired number of worker replicas is now increased to 40. Many of them are currently being created. Now you might be wondering, what happen if we run out of CPUs. After all, each worker requires 1 cpu to run. 
1. We can take a look and see how our computing resource is doing by typing `kubectl get node` into our terminal.
1. To see if we have exhausted a ll the CPU or memory resources, we can use `kubectl describe ` follow by the name of the node to check the CPU usage. 
1. By looking the the results, we can see that our CPU is about maxed out. And further more we can see that our machine only has 32 cpu cores
1. If we need more computing resource, we can use a GCP command ask for more computing resource, by increasing the size of our node pool. `gcloud container clusters resize {name of project} --node-pool={name of pool} --num-nodes={number of desired nodes}`. I'm going to scale to 4 machines.
1. This will again enabled us to increase the number of workers to about 120, remember that each worker is equal to 1 cpu.
1. This means we now have 120 CPU cores doing the parameter tuning for us
1. Essentially we can keep scaling out the number of machines, and the number of worker nodes, with enough money that is.
1. So this is how we'd scale parameter tuning to hundreds of CPUs, using Kubernetes and a lot of infrastructure code. This completes our first video. In the next video, we'll go into more details what each yaml file does.