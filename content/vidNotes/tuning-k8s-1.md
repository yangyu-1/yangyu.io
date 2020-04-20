In this second video we'll dive into the details of our rabbitMQ and worker deployments

Let's get started:
1. We'll take a look at the rabbitMaster.yaml file. I've made some changes to the labels from last video to better clarify what some of the fields are doing. 
1. Inside this file, there are two Kubernetes resources, separated by the triple dash in line 23. The first object is a deployment resource, and the second is a service resource. 
1. The deployment object is a higher level Kubernetes resource that manages Replicasets which in turn manage pods. Pods represent the atomic unit of work in a Kubernetes cluster, it can comprise one or more containers. Pods are ephemeral in nature, meaning they can go away due to reasons like migration, or node failure.
1. To maintain uptime, Pods are usually manage by a higher level resources like Deployment or ReplicaSets. These higher level resource combine Pod creation and desired numb er of replicas into a unifying API. 

1. Let's first look at the Pod creation API; In our deployment specs, fields under the template specifies how the pod is going to be created, for example, all of the pod created by our deployment will have the labels app: param-tuning and type: message-queue-pod. Type in the command `kubectl get po --show-labels` to show the pods, you can see the labels correctly matches the yaml.
1. You might have noticed there are two metadata field in the Deployment spec. the first metadata applies to the deployment itself, while the one inside template applies to the Pods. Command `kubectl get deploy --show-labels` will show the labels for our deployment, and as expected they correspond to the first metadata field.
1. A easy way to remember this behavior is just to remember that template, and only template applies to the pods created by the deployment, while everything else applies to the deployment itself.

1. Inside our pod, we're creating one container by pulling the image rabbitmq:3-management, and naming it rabbitmq-container. We also open two ports. 5672 is the default port for communication with rabbitmq, and 15672 is the management port. 
1. NodeSelector means that the pod will only be schedule to nodes that are part of the pool name "default-pool"

1. That completes our Pod creation part of the yaml file. Now let's look at two fields that acts on the Pods created.
1. The selector field defines how the Deployment finds which Pods to manage. In this case, we simply select a label that is defined in the Pod template.
1. The replicas field indicates how many copies of the pod will be running. Here we only have 1 copy.

1. Next let's look at the Service object.
1. A service is a way to "expose an application running on a sets of pods as a network service"
1. Since our workers and message queue do not share a network space. Without service, our workers wouldn't be able to communicate with our message queues.
1. Service objects are relatively simple to create. There are two key parts
1. One of them is the selector field. This field means that all pods with the selected label will be part of this service.  The label here must match the label we assigned to our pods in the template field.
1. Next we see two ports associated with this service. This means that any traffic that we sent to rabbitmq-service at port 5672, would be forwarded to the Pods' port 5672. Target port must match container port. But we can change the port field. For example, if we change port to 6000. Then our rabbitmq service will be available at rabbitmq-service:6000. And any traffic to that URL will be forwarded to the pod's 5672 port.
1. There are multiple way s to validate if the Service resource correctly connects to our Pods. You can create a temporary Pod inside the cluster to curl or ping the service to see if it is up and running. Or you can check if the endpoints are connected to the right pods. Here I'll sho  w the second method, but for a more comprehensive debugging guide, I linked a document call Debugging Services from the Kubernetes official website in the description.
1. First run the command `kubectl get po -o=wide` to get the IP of the POD.
1. Next run `kubectl get endpoints rabbitmq-service` to get the endpoints for the service. The IP address should match the Pods IP, meaning that the service is sending the data to the correct Pod.

1. Now let's take a look at our workers. Remember, our workers is a python program that reads the message in our message queue, does some processing, and saves the result to a mongodb database. 
1. In order to run our workers, we must first create the container image. I'm not going to go over how to create that, if you're interested, both the code, and the dockerfile in the worker folder. 
1. Once you build the image, you can push it to any public or private docker registries, and configure kubernetes to pull from them. By default, kubernetes pulls public images from dockerHub. 
1. I do want to point out one thing in the code, which will lead into our discussion of the yaml file. In the worker.py file, we see that our worker is trying to connect to our messageQueue using a environmental variable call secretURL. 
1. This environmental variable is coming from a configMap resource call prod-config. configMap is a Kubernetes resource that provides key-value pairs that can be used configure containers. In our example, our worker Pod reads the secretURL value from the configMap, which matches to the Service we created in rabbitmq-service. Of course we can hardcode the rabbitmq-service address inside of our code, but that would make it harder for us to reuse the container image. 
1. Next, unlike our rabbitMQ deployment, we specified the amount of CPU and RAM our workers are allowed to consume. We specified both requests and limits for resources, meaning the container will be guaranteed 1 CPU core and 750Mi of RAM
1. Finally, the workers are deploy to a pool call pool-1, as oppose to default-pool, which is where rabbitMQ is deployed. If we take a look at my GCP project, we can see that the two pools utilize different machine types, and that pool-1 utilizes GCP's preemptive nodes to save code. If the node fails during a training run, the pods will die but the training task will be returned to queue, meaning it wouldn't affect our overall system. However, if the message queue or the database goes down, it will bring the whole system down.