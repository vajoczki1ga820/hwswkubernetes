# Lecture summary: 
- Difference between a container and a Pod
- ReplicaSet
- Deployment
- Deployment history and rollout strategies
- DaemonSet
- Job
- CronJob

# Recordings:
- [1/4](https://hwsw.adobeconnect.com/p91jqbk9zxql/)
- [2/4](https://hwsw.adobeconnect.com/pa104kgqkx1d/)
- [3/4](https://hwsw.adobeconnect.com/pa8ecky9uryv/)
- [4/4](https://hwsw.adobeconnect.com/p80sqcd2tofr/)

## Set autocompletion for kubectl command:

    # setup autocomplete in bash into the current shell
    source <(kubectl completion bash)

    # add autocomplete permanently to your bash shell.
    echo "source <(kubectl completion bash)" >> ~/.bashrc 

## Demo 1: Imperative vs. Declarative Commands
    
There are 3 ways of handling a resource (here Pod):
    
1. imperative way: intent is on the command

        # create pod
        kubectl run www --image=nginx:1.16
        # check
        kubectl get pods -o wide
        kubectl delete pod www
        kubectl get pods -o wide
        
        # Further examples:
        kubectl label pod www app=frontend
        kubectl get pod www --show-labels
        kubectl label pod www app-
        kubectl annotate pod www myapp=alma
        kubectl expose pod www --port 80 --target-port 8080 
    
2. imperative object config management: intent is on the command, but we also have a yaml file
    
        # dry run for create a resource definition 
        kubectl run www --image=nginx:1.16 --dry-run=client -o yaml > nginx-pod.yaml
        cat nginx-pod.yaml
        
        # create from file
        kubectl create -f nginx-pod.yaml
        kubectl get pod www -o yaml
        
        # equivalent commands:
        kubectl delete -f nginx-pod.yaml
        kubectl delete pod www
    
3. declarative way: no intent is in the command, K8s finds out what needs to be done (reconciliation)
    
        # dry run for create a resource definition 
        kubectl run www --image=nginx:1.16 --dry-run=client -o yaml > nginx-pod.yaml
        cat nginx-pod.yaml
        
        # let k8s finds out what to do (at this point "create" or "configure" both possible options)
        kubectl apply -f nginx-pod.yaml
        
        # labeling, changing container version, etc.: 
        nano nginx-pod.yaml
        
        # this here is definitely a "configure" action
        kubectl apply -f nginx-pod.yaml
        
        kubectl delete -f nginx-pod.yaml

## Demo 2: Namespaces

    kubectl create namespace test
    kubectl get ns
    kubectl -n test get pods -o wide
    
    # add namespace to pod definition
    nano nginx-pod.yaml
    kubectl create -f nginx-pod.yaml
    kubectl -n test get pods -o wide

    # deleting namespace does delete everything inside it
    kubectl delete ns test
    kubectl -n test get pods -o wide

    # create test namespace again
    kubectl create namespace test
    kubectl -n test get pods -o wide

    # set kubectl context to "test" namespace as everything will be created there
    kubectl config set-context --current --namespace=test
    
    # clean up in test namespace: 
    kubectl delete all --all

## Demo 3: Multicontainers: 
    
pause container:

    docker ps # on where the pod will be scheduled
    kubectl run www --image=nginx:1.16
    
    # check where the pod is running
    kubectl get pods -o wide
    docker ps # to where the pod is scheduled
    
app containers:

    cat multicontainer-application-pod.yaml
    kubectl apply -f multicontainer-application-pod.yaml
    kubectl get pods -o wide
    curl <POD-IP>
    
init container:

    cat multicontainer-init-pod.yaml
    kubectl apply -f multicontainer-init-pod.yaml
    kubectl get pods -o wide
    
    # after pod is running check logs (stdout)
    kubectl logs multicontainer-init

    # both containers can be noticed 
    kubectl describe pod multicontainer-init
    
clean up in test namespace:

    kubectl delete all --all 

## Demo 4: Environments that a container see in the Pod

    kubectl run nginx --image=nginx:1.16

    # get inside the container
    kubectl exec -it nginx -- bash
    
    # check what we can see from the inside
    printenv
    ls
    hostname
    
    # clean up in test namespace: 
    kubectl delete all --all 

## Demo 5: ReplicaSet
    
create ReplicaSet:

    cat replicaSet-less-pod.yaml
    cat replicaSet.yaml
    
    # create one pod with the same labels that are in the replicaSet selectors
    kubectl apply -f replicaSet-less-pod.yaml
    
    # create replicaSet and notice that for the 3 replicas it creates only 2 new ones
    # since one replica already exists (the separately created pod)
    kubectl apply -f replicaSet.yaml
    kubectl get pods -o wide
    kubectl get replicasets
    kubectl describe replicasets frontend

self healing and scaling:

    kubectl delete pod <TAB>
    # change replicas: 2 --> youngest Pod will be terminated
    nano replicaSet.yaml

    # notice that now we scale down the pods
    kubectl apply -f replicaSet.yaml 
    kubectl get pods -o wide

    kubectl delete -f replicaSet.yaml
    # check that all pods are deleted (even the one created separately)
    kubectl get pods -o wide
  
selector decides which Pod belongs to the ReplicaSet:

    kubectl get pods -o wide
    kubectl apply -f replicaSet-less-pod.yaml
    kubectl apply -f replicaSet.yaml 
    kubectl get pods -o wide
  
manipulating Pod labels:

    kubectl get pods -o wide --show-labels

    # multiple ways to 
    KUBE_EDITOR="nano" kubectl edit pod <TAB>
    kubectl get pods -o wide --show-labels
    kubectl label pod <POD-NAME> app2=nginx app-
    kubectl label pod <POD-NAME> tier=frontend

    # add label tier=frontend2 to pod(s) with label tier=frontend
    kubectl label pod -l tier=frontend tier=frontend2 --overwrite

clean up in test namespace: 

    kubectl delete all --all 

## Demo 6: Deployment
  
introducing new label -> pod-template-hash:

    cat deployment.yaml
    kubectl apply -f deployment.yaml

    # notice that k8s has created three objects
    kubectl get deployment --show-labels
    kubectl get rs --show-labels
    kubectl get pod --show-labels
    
revisions:

    kubectl rollout history deployment frontend-deployment
    # scaling -> not change revision number, kubectl EDIT or nano+apply are also options
    kubectl scale deployment frontend-deployment --replicas=2
    # check full history
    kubectl rollout history deployment frontend-deployment
    # check specific revision
    kubectl rollout history deployment frontend-deployment --revision=1
      
    # scaling -> can be recorded but still not change revision number 
    kubectl scale deployment frontend-deployment --replicas=2
    kubectl rollout history deployment frontend-deployment
  
    # revision history limit
    kubectl get deployments frontend-deployment -o yaml | grep -i "revisionHistory"
  
    # change pod template -> nginx version
    nano ...
    kubectl apply -f deployment.yaml
    kubectl rollout history deployment frontend-deployment
    kubectl rollout history deployment frontend-deployment --revision=2
    # check nginx version
    kubectl describe pod <TAB>

    # set back to revision one (in this case `--to-revision=1` is not necessary)
    kubectl rollout undo deployment frontend-deployment --to-revision=1
    kubectl get pods -o wide
    # check nginx revision
    kubectl get deployments frontend-deployment -o yaml
    
    # check ReplicaSets
    kubectl get rs --show-labels
    kubectl get pod --show-labels
      
    # pause/resume
    kubectl rollout pause deployment frontend-deployment
    
    # only describe shows the paused state
    kubectl get deploy
    kubectl rollout status deployment frontend-deployment
    kubectl describe deployment frontend-deployment # Progressing Unknown DeploymentPaused
    
    # change image and replicas
    kubectl set image deployment frontend-deployment nginx=busybox:1.16 --record
    kubectl scale deployment frontend-deployment --replicas=1
    kubectl rollout resume deployment frontend-deployment

    # clean up in test namespace: 
    kubectl delete all --all

## Demo 7: Deployment Update strategy: rolling update
    
    # there are 15 replicas with image version 1.0
    cat deployment.yaml
    kubectl apply -f rollout-deployment.yaml
    # check how rs replica counter
    kubectl get rs -o wide
    
    # change image version to 2.0
    nano rollout-deployment.yaml
    kubectl apply -f rollout-deployment.yaml
    # check how rs replica counter
    watch kubectl get rs -o wide

    # clean up in test namespace: 
    kubectl delete all --all

## Demo 8: Service
  
    # lonley pod 
    cat replicaSet-less-pod.yaml
    kubectl apply -f replicaSet-less-pod.yaml
    curl <POD-IP>:80

    # let's create Service for our lonely pod
    cat service.yaml
    kubectl apply -f service.yaml
    kubectl get service -o wide
    curl <SERVICE-IP>:8080
    # service finds the pod(s) similarly to a replica set
    kubectl get endpoints my-service
  
    # DNS won't work, only from PODs
    curl my-service:8080
    kubectl run curl --image=radial/busyboxplus:curl -i --tty
    # curl inside the busybox pod
    curl my-service:8080
  
    # delete pod and check what happens
    kubectl delete pod <TAB>
    curl <SERVICE-IP>:8080
    kubectl get pod -o wide
    kubectl get endpoints my-service
    
    # create deployment with service (the most common thing in K8s)
    cat service-deployment.yaml
    kubectl apply -f service-deployment.yaml
    
    # edit service.yaml to have the same label selector
    # this is a ClusterIP type service
    nano service.yaml
    kubectl apply -f service.yaml
    kubectl get endpoints my-service

    # clean up in test namespace: 
    kubectl delete all --all 

## Demo 9: Create multiport service with defined cluster IP

    cat multiport-cluster-external-ip-service.yaml
    kubectl apply -f multiport-cluster-external-ip-service.yaml
    kubectl get service

    # clean up in test namespace: 
    kubectl delete all --all  

## Demo 10: NodePort service

    cat nodeport-service.yaml
    kubectl apply -f nodeport-service.yaml
    # notice the ports
    kubectl get svc
    curl <SERVICE-IP>:8080
    curl <NODE-IP>:30007

    # clean up in test namespace: 
    kubectl delete all --all 

## Demo 11: ExternalName Service

    cat externalName-service.yaml
    kubectl apply -f externalName-service.yaml
    kubectl run curl --image=radial/busyboxplus:curl -i --tty
    nslookup my-service
    # its really a Google served webpage
    curl my-service

  # clean up in test namespace: 
    kubectl delete all --all 

## Demo 12: DNS normal service, headless service, pod 

    cat dns-nginx-deployment.yaml
    kubectl apply -f dns-nginx-deployment.yaml
    cat nginx-normal-service.yaml
    kubectl apply -f  nginx-normal-service.yaml
    cat nginx-headless-service.yaml
    kubectl apply -f nginx-headless-service.yaml
    kubectl get all
    kubectl run busy --image=busybox:1.27 -i --tty
    nslookup normal-service
    nslookup headless-service
    nslookup <POD-IP>.default.pod

    # clean up in test namespace: 
    kubectl delete all --all 

## Demo 13: DaemonSet

    # log collection example with fluentd
    cat daemonset.yaml
    kubectl apply -f daemonset.yaml
    kubectl get pods -o wide
    kubectl logs <POD-NAME> 
  
    # clean up in test namespace: 
    kubectl delete all --all   

## Demo 14: Static pod

    systemctl status kubelet
    # check config file path: --config=/var/lib/kubelet/config.yaml
    # check for statiPodPath:
    cat /var/lib/kubelet/config.yaml | grep -i staticPod
    #output `staticPodPath: /etc/kubernetes/manifests/`
    cp static-pod.yaml /etc/kubernetes/manifests/static-pod.yaml
    kubectl get pods -o wide
    curl <POD-IP> 
    # try to delete it
    kubectl delete pod <POD-NAME>
    # check 
    kubectl get pods -o wide
    # delete
    rm /etc/kubernetes/manifests/static-pod.yaml
    # check 
    kubectl get pods -o wide

    # clean up in test namespace: 
    kubectl delete all --all 

## Demo 15: Job

    cat job.yaml
    kubectl apply -f job.yaml
    kubectl get pods -o wide
    kubectl get job
  
    # clean up in test namespace: 
    kubectl delete all --all 

## Demo 16: CronJob
    
    cat cronjob.yaml
    kubectl apply -f cronjob.yaml
    kubectl get pods -o wide
    watch kubectl get cronjob
    kubectl get pods -o wide
    kubectl get job 
    kubectl get cronjob
  
    # clean up in test namespace: 
    kubectl delete all --all
