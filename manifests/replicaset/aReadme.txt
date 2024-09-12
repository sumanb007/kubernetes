kubectl get replicaset
or 
kubectl get rs
or
kubectl get rs <name-of-replicaset>
	=> check how many replicasets in system

kubectl describe replicaset <name-of-replicaset>
	=> check the status of replicaset

kubectl edit replicaset <name-of-replicaset>
	=> to edit replicaset when pods are created already

kubectl scale replicaset <name-of-replicaset> --replicas=2
	=> manual method to scale replicaset

kubectl delete rs <name-of-replicaset>
	=> delete replicaset

