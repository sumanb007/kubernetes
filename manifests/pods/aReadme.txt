kubectl create pod --help


kubectl run mypod --image=nginx
	=> imperative command to simply crate pod using image


kubectl scale deployment httpd-frontend --replicas=5
	=> imperative command to scale a deployment


kubectl expose deployment httpd-frontend --port=80 --type=NodePort
	=> imperative command to expose a service


kubectl delete pod mypod
	=> to delete resource



kubectl delete pod nginx-deployment-858df98448-z4zkv --grace-period=0 --force
	=> deleting stubborn pods forcefully
or bash script
by using bash script for all pods in deployment
--> for pods in $(kubectl get pods | grep 'Terminating' | awk '{print $1}'); do kubectl delete pod $pods --grace-period=0 --force ; done




kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
	=> to produce declarative configurations




*********  Understanding --dry-run Modes ************************************

1. --dry-run=client (or just --dry-run in older versions):

Example:
	kubectl run nginx --image=nginx --dry-run=client -o yaml
		=> This command will output the YAML configuration of the Pod that would be created without actually creating it.


This mode simulates the command locally on the client side without sending any request to the API server.

It is used to validate the command and see what would be sent to the server without making any actual changes to the cluster.

You can use this mode to generate manifest files or to preview the resource that would be created.





2. --dry-run=server:

Example:
	kubectl run nginx --image=nginx --dry-run=server
	=> This validates the command against the server-side policies but doesn't create any resources.


This mode sends the request to the API server to validate it, but does not persist the changes.

This checks against the actual state and server-side constraints like admission controllers, validating webhooks, and quota policies. 

It provides a more accurate validation, considering all server-side rules and configurations.




3. --dry-run=none (Default):

This is the default mode if --dry-run is not specified, where the command is fully executed, and resources are created as specified.


IN GENERAL

* Use client: When you want to quickly validate or generate manifests on the client side without involving the server.

* Use server: When you need to validate the command considering the server-side context, policies, and configurations.

* Use none (or omit --dry-run): When you want to apply the command and create resources in the cluster.


