kubectl get deploy
or
kubectl get deployment
	=> to check the status of deployments

kubectl edit deploy <name-of-deployment>
	=> to edit the deployment pods at once

kubectl set image deployment/faulty-deployment faulty-container=nginx:latest
	=> imperative command to fix the issue in container image


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


kubectl rollout status deployment/nginx-deployment
	=> checking the rollout status


kubectl rollout history deployment/nginx-deployment
	=> checking the rollout history


kubectl rollout undo deployment/nginx-deployment --to-revision=1
or
kubectl rollout undo deployment/nginx-deployment
	=> rollback to desired revision

kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="Rollback to revision-1
	=> recording the cause of rollback


kubectl get events --field-selector involvedObject.name=nginx-recreate --output=json
	=> This command retrieves events related to the resource named my-nginx.



kubectl get events --sort-by='.metadata.creationTimestamp'
	=> Sorts the kubernetes events based on time
