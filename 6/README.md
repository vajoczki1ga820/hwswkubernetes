## Set autocompletion for kubectl command:

    # setup autocomplete in bash into the current shell
    source <(kubectl completion bash)

    # add autocomplete permanently to your bash shell.
    echo "source <(kubectl completion bash)" >> ~/.bashrc 


## Demo 0: recap the previous learings
	
	# run a complete microservice environment with deployments and services
	# https://github.com/microservices-demo/microservices-demo
	kubectl create namespace sock-shop

	# on Kubernetes version 1.16+ run this
	kubectl apply -f https://raw.githubusercontent.com/rexx4314/microservices-demo/patch-1/deploy/kubernetes/complete-demo.yaml 
	# on Kubernetes version <1.16 run this
	kubectl apply -f https://raw.githubusercontent.com/microservices-demo/microservices-demo/master/deploy/kubernetes/complete-demo.yaml
	
	# see the created pods
	kubectl get pods -n sock-shop

	# see the created services
	kubectl get services -n sock-shop
	
	# create an ingress for the front-end services
	cat sock-shop-ingress.yaml
    kubectl apply -f sock-shop-ingress.yaml
	kubectl get ingress -n sock-shop
	
	# log in via the front-end with user/password, create an order, then delete the DB of the orders services
	kubectl delete -n sock-shop pod $(kubectl get pods -n sock-shop -o name | grep orders-db | sed 's/.*\///')
	kubectl get pods -n sock-shop
	
	# clean-up the demo
	kubectl delete ns sock-shop

## Demo 1: emptydir volume example
  
    cat emptydir-pod.yaml
    kubectl apply -f emptydir-pod.yaml
    # check the node where it is placed
    kubectl get pods -o wide
    # print data stored in cache volume
    kubectl logs <POD-NAME>
    # check the app container 
    sudo docker ps -a | grep busy
    # remove the app container
    sudo docker rm -f <CONTAINER-ID>
    # notice that another one is started
    sudo docker ps -a | grep busy
    # notice that the pod is still running
    kubectl get pods -o wide
    # notice that the data persisted even when the container was deleted
    kubectl logs <POD-NAME>
  
## Demo 2: downward API volume example

    cat downward-api-pod.yaml
    kubectl get pods -o wide
    # check the pod information inside the container
    kubectl exec kubernetes-downwardapi-volume-example -- cat /etc/podinfo/labels
    kubectl exec kubernetes-downwardapi-volume-example -- cat /etc/podinfo/annotations
    
## Demo 3: manual persistent volume creation example

    cat fake-nfs-pv.yaml
    kubectl apply -f fake-nfs-pv.yaml
    # notice status (available)
    kubectl get pv
    
    cat fake-nfs-pvc.yaml
    kubectl apply -f fake-nfs-pvc.yaml
    # notice status (bound)
    kubectl get pvc
    # notice status (bound)
    kubectl get pv
    
    cat fake-nfs-pvc-pod.yaml
    kubectl apply -f fake-nfs-pvc-pod.yaml
    # check where the pod is placed and where the pvc is defined 
    # they need to be on the same node 
    kubectl get pods -o wide 
    # on the node where the pod is
    echo "some content" > /tmp/nfs/data/bigdata
    # check the content in the container
    kubectl exec pvc-pod-example -- cat /tmp/data/bigdata
  
## Demo 4: Dynamic storage provision with storage class

    # for this to work Google managed Kubernetes cluster (GKE) is needed
    
    # check GKE storage classes
    kubectl get storageclasses.storage.k8s.io
    # take a closer look on it (type: pd-standard --> slow type, not ssd)
    kubectl get storageclasses.storage.k8s.io standard -o yaml
    # since it already exits we just need to refer to it with a pvc
    cat dynamic-pvc.yaml
    # before we create it, check existing pv-s
    kubectl get pv -o wide
    kubectl apply -f dynamic-pvc.yaml
    kubectl get pv
    kubectl get pvc
    
    # create the pod that will use the pvc
    cat dynamic-pvc-pod.yaml
    kubectl apply -f dynamic-pvc-pod.yaml
    kubectl get pods -o wide
    
    kubectl delete pod <POD-NAME> 
    kubectl delete pvc <PVC-NAME>
    # notice that PV is automatically deleted as well 
    # because in the standard storage class: "reclaimPolicy: Delete"
    kubectl get pv
    
## Demo 5: Longhorn
	# install Longhorn
	kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
	kubectl get pods -n longhorn-system
	kubectl patch -n longhorn-system service longhorn-frontend -p '{"spec":{"type":"NodePort"}}'
	kubectl get services -n longhorn-system
	
	# view the storageclass
	kubectl get storageclasses.storage.k8s.io
	
	# make it the default one
	kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
	kubectl get storageclasses.storage.k8s.io

## Demo 6: StatefulSet
	# create a statefulset for Zookeeper
	cat zookeeper-deployment.yaml
	kubectl apply -f zookeeper-deployment.yaml
	kubectl get pods,pvc,pv
	
	# view the PersistentVolumeClaims and PersistentVolumes
	kubectl get pv,pvc
	
	# create a headless service
	cat zookeeper-service.yaml
	kubectl apply -f zookeeper-service.yaml
	kubectl get service
	
	# examin the DNS names and IP addresses
	kubectl exec -it zookeeper-0 -- nslookup zookeeper
	
## Demo +1: Horizontal Pod Autoscaler example

    cat hpa-deployment.yaml
    kubectl apply -f hpa-deployment.yaml
    kubectl get pods -o wide

    # create HPA
    kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
    kubectl get hpa

    # increase the load
    kubectl run -it --rm load-generator --image=busybox /bin/sh
    # run command
    while true; do wget -q -O- http://php-apache; done

    # check pods load and how autoscaler reacts:
    kubectl top pods
    kubectl get hpa

    # stop load
    CTRL + C in the container

    # check pods load and how autoscaler reacts (should take a few seconds to scale down the pods)
    kubectl top pods
    kubectl get hpa

    # clean up in test namespace: 
    kubectl delete all --all
    kubectl delete hpa php-apache