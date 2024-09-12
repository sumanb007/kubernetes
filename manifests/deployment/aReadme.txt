kubectl get deploy
or
kubectl get deployment
	=> to check the status of deployments

kubectl edit deploy <name-of-deployment>
	=> to edit the deployment pods at once

kubectl describe deploy <name-of-deployment>
	=> to check the status of all deployment pods

kubectl scale deploy <name-of-deployment> --replicas=<number of replicas>
	=> manual method to scale deployment

kubectl delete deploy --all
	=> delete all deployments in system


kubectl create deployment httpd-frontend --replicas=3 --image=httpd:2.4-alpine --dry-run -o yaml
	=> create declarative yaml file for deployment

kubectl create deploy --help
	=> to create deployment without composing manifest file
example:
	kubectl create deployment httpd-frontend --replicas=3 --image=httpd:2.4-alpine
