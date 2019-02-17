  
KUBERNETES:
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Creating a Kubernetes cluster using minikube:-
----------------------------------------------
-> minikube version 
It gives the minikube version: v0.28.2

-> minikube start
This command will start the local kubernetes,VM and cluster components.

-> kubectl version
This command will give the client and server version of kubectl.

-> kubectl cluster-info   
Kubernetes master is running at https://172.17.0.12:8443

-> kubectl get nodes
This command shows all nodes that can be used to host our applications and the status.
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    2m        v1.10.0

Deploying an application on Kubernetes using kubectl:-
------------------------------------------------------
-> kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080
deployment.apps/kubernetes-bootcamp created
i.e., the run command will create a new deployment by taking the deployment name and image location.It requires port parameter to deploy on a specific port.

-> kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            1           6m
i.e., this command will list the deployments.

-> kubectl proxy
When we use kubectl, we're interacting through an API endpoint to communicate with our application.
The kubectl command can create a proxy that will forward communications into the cluster-wide, private network. 


-> export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{end}}'
   echo Name of the Pod: $POD_NAME
Name of the Pod: kubernetes-bootcamp-5c69669756-zlgtl

-> curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-5c69669756-zlgtl | v=1
To make an HTTP request to the application running in that pod.

Explore an application:-
------------------------

To get list of pods
-> kubectl get pods

Next, to view what containers are inside that Pod and what images are used to build those containers we run the describe pods command:
-> kubectl describe pods

Run poxy
-> kubectl proxy

Now again, we'll get the Pod name and query that pod directly through the proxy. To get the Pod name and store it in the POD_NAME environment variable:
-> export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
   echo Name of the Pod: $POD_NAME

To see the output of our application, run a curl request.
-> curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/


Anything that the application would normally send to STDOUT becomes logs for the container within the Pod. We can retrieve these logs using the kubectl logs command:
-> kubectl logs $POD_NAME

We can execute commands directly on the container once the Pod is up and running. For this, we use the exec command and use the name of the Pod as a parameter. Let’s list the environment variables:
-> kubectl exec $POD_NAME env


Next let’s start a bash session in the Pod’s container:
-> kubectl exec -ti $POD_NAME bash

We have now an open console on the container where we run our NodeJS application. The source code of the app is in the server.js file:
-> cat server.js

You can check that the application is up by running a curl command:
-> curl localhost:8080

To close your container connection type exit
-> exit

Exposing Your Application:-
---------------------------
Let’s verify that our application is running. We’ll use the kubectl get command and look for existing Pods:
-> kubectl get pods

Next let’s list the current Services from our cluster:
-> kubectl get services

We have a Service called kubernetes that is created by default when minikube starts the cluster. To create a new service and expose it to external traffic we’ll use the expose command with NodePort as parameter (minikube does not support the LoadBalancer option yet).
-> kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080

Let’s run again the get services command:
-> kubectl get services

We have now a running Service called kubernetes-bootcamp. Here we see that the Service received a unique cluster-IP, an internal port and an external-IP (the IP of the Node).
To find out what port was opened externally (by the NodePort option) we’ll run the describe service command:
-> kubectl describe services/kubernetes-bootcamp

Create an environment variable called NODE_PORT that has the value of the Node port assigned:
-> export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
   echo NODE_PORT=$NODE_PORT

Now we can test that the app is exposed outside of the cluster using curl, the IP of the Node and the externally exposed port:
-> curl $(minikube ip):$NODE_PORT

The Deployment created automatically a label for our Pod. With describe deployment command you can see the name of the label:
-> kubectl describe deployment

Let’s use this label to query our list of Pods. We’ll use the kubectl get pods command with -l as a parameter, followed by the label values:
-> kubectl get pods -l run=kubernetes-bootcamp

You can do the same to list the existing services:
-> kubectl get services -l run=kubernetes-bootcamp

Get the name of the Pod and store it in the POD_NAME environment variable:
-> export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
   echo Name of the Pod: $POD_NAME

To apply a new label we use the label command followed by the object type, object name and the new label:
-> kubectl label pod $POD_NAME app=v1

This will apply a new label to our Pod (we pinned the application version to the Pod), and we can check it with the describe pod command:
-> kubectl describe pods $POD_NAME

We see here that the label is attached now to our Pod. And we can query now the list of pods using the new label:
-> kubectl get pods -l app=v1

To delete Services you can use the delete service command. Labels can be used also here:
-> kubectl delete service -l run=kubernetes-bootcamp

Confirm that the service is gone:
-> kubectl get services

This confirms that our Service was removed. To confirm that route is not exposed anymore you can curl the previously exposed IP and port:
-> curl $(minikube ip):$NODE_PORT

This proves that the app is not reachable anymore from outside of the cluster. You can confirm that the app is still running with a curl inside the pod:
-> kubectl exec -ti $POD_NAME curl localhost:8080

We see here that the application is up
To list your deployments use the get deployments command: 
-> kubectl get deployments

The DESIRED state is showing the configured number of replicas
The CURRENT state show how many replicas are running now
The UP-TO-DATE is the number of replicas that were updated to match the desired (configured) state
The AVAILABLE state shows how many replicas are actually AVAILABLE to the users
Next, let’s scale the Deployment to 4 replicas. We’ll use the kubectl scale command, followed by the deployment type, name and desired number of instances:
-> kubectl scale deployments/kubernetes-bootcamp --replicas=4

Scaling Your Application:-
--------------------------
To list your Deployments once again, use get deployments:
-> kubectl get deployments

The change was applied, and we have 4 instances of the application available. Next, let’s check if the number of Pods changed:
-> kubectl get pods -o wide

There are 4 Pods now, with different IP addresses. The change was registered in the Deployment events log. To check that, use the describe command:
-> kubectl describe deployments/kubernetes-bootcamp

You can also view in the output of this command that there are 4 replicas now.
Let’s check that the Service is load-balancing the traffic. To find out the exposed IP and Port we can use the describe service as we learned in the previously Module:
-> kubectl describe services/kubernetes-bootcamp

Create an environment variable called NODE_PORT that has a value as the Node port:
-> export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
   echo NODE_PORT=$NODE_PORT

Next, we’ll do a curl to the exposed IP and port. Execute the command multiple times:
-> curl $(minikube ip):$NODE_PORT

We hit a different Pod with every request. This demonstrates that the load-balancing is working.
To scale down the Service to 2 replicas, run again the scale command:
-> kubectl scale deployments/kubernetes-bootcamp --replicas=2

List the Deployments to check if the change was applied with the get deployments command:
-> kubectl get deployments

The number of replicas decreased to 2. List the number of Pods, with get pods:
-> kubectl get pods -o wide

This confirms that 2 Pods were terminated

Updating Your Application:-
---------------------------
To list your deployments use the get deployments command: 
-> kubectl get deployments

To list the running Pods use the get pods command:
-> kubectl get pods

To view the current image version of the app, run a describe command against the Pods (look at the Image field):
-> kubectl describe pods

To update the image of the application to version 2, use the set image command, followed by the deployment name and the new image version:
-> kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2

The command notified the Deployment to use a different image for your app and initiated a rolling update. Check the status of the new Pods, and view the old one terminating with the get pods command:
-> kubectl get pods

First, let’s check that the App is running. To find out the exposed IP and Port we can use describe service:
-> kubectl describe services/kubernetes-bootcamp

Create an environment variable called NODE_PORT that has the value of the Node port assigned:
-> export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
   echo NODE_PORT=$NODE_PORT

Next, we’ll do a curl to the the exposed IP and port:
-> curl $(minikube ip):$NODE_PORT

The update can be confirmed also by running a rollout status command:
-> kubectl rollout status deployments/kubernetes-bootcamp

To view the current image version of the app, run a describe command against the Pods:
-> kubectl describe pods

Let’s perform another update, and deploy image tagged as v10 :
-> kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10

Use get deployments to see the status of the deployment:
-> kubectl get deployments

And something is wrong… We do not have the desired number of Pods available. List the Pods again:
-> kubectl get pods

A describe command on the Pods should give more insights:
-> kubectl describe pods

There is no image called v10 in the repository. Let’s roll back to our previously working version. We’ll use the rollout undo command:
-> kubectl rollout undo deployments/kubernetes-bootcamp

The rollout command reverted the deployment to the previous known state (v2 of the image). Updates are versioned and you can revert to any previously know state of a Deployment. List again the Pods:
-> kubectl get pods

Four Pods are running. Check again the image deployed on the them:
-> kubectl describe pods

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------






