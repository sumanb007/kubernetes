kubectl get service
or
kubectl get svc
or
kubectl get svc <service-name>
	=> to get status of service


kubectl describe svc kubernetes
	=> to check the detail of specified service


kubectl get endpoints
	=> to get status of the 'pods IPs' or endpoints


kubectl expose deployment httpd-deployment --type=ClusterIP --port=80 --target-port=80 --name=httpd-service
	=> imperative command to create service


kubectl expose deployment httpd-deployment --type=ClusterIP --port=80 --target-port=80 --name=httpd-service --dry-run=client -o yaml > httpd-service.yaml
	=> imperative command to create manifest file for service
