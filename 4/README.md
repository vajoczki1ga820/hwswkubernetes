# Lecture summary: 

- Args and environment variables
- ConfigMap and Secret
- Limit and Request
- Scheduling (affinity, anti-affinity, taints)
- Troubleshooting basics

# Recordings:
- [1/3](https://hwsw.adobeconnect.com/pw8y84wgh4t8/)
- [2/3](https://hwsw.adobeconnect.com/p01zh58egae7/)
- [3/3](https://hwsw.adobeconnect.com/pgbm0ccjlqfy/)

## Set autocompletion for kubectl command

    # setup autocomplete in bash into the current shell
    source <(kubectl completion bash)

    # add autocomplete permanently to your bash shell.
    echo "source <(kubectl completion bash)" >> ~/.bashrc 

## Create test namespace in which we run the examples

    # create test namespace 
    kubectl create namespace test

    # set kubectl context to "test" namespace as everything will be created there
    kubectl config set-context --current --namespace=test
    
## Demo 1: Resources

    # limits: 0.5 < CPU < 1, but in the argument we start to use 2 CPUs 
    cat resource-pod.yaml
    kubectl apply -f resource-pod.yaml
    kubectl get pods -o wide
    top # on the node it runs
    # delete pod, change request and limit to 100 for CPU, create it
    kubectl delete -f resource-pod.yaml
    kubectl apply -f resource-pod.yaml
    # notice the pending state. The request is so huge that k8s cannot even start the pod
    kubectl get pods -o wide
    kubectl describe pod cpu-demo
    
    # clean up in test namespace: 
    kubectl delete all --all

## Demo 2: LivenessProbe and readinessProbe

    cat livenessProbe-exec-pod.yaml
    # command/script based health check -> should fail in every ~25 sec
    kubectl apply -f livenessProbe-exec-pod.yaml
    watch kubectl get pods -o wide
    
    
    # API based check 
    cat livenessProbe-http-pod.yaml
    # code in: [200, 400) -> success, anything else -> failure
    # server sends 500 after 10 seconds -> should restart in every ~20 sec
    kubectl apply -f livenessProbe-http-pod.yaml
    watch kubectl get pods -o wide

    # readiness probe example
    # since the application need at least 10 sec (for create ready file)
    # we set readiness delay at 15 sec
    cat readinessProbe-exec-pod.yaml
    # very similar possibilites to liveness probes
    kubectl apply -f readinessProbe-exec-pod.yaml
    watch kubectl get pods -o wide
  
    # clean up in test namespace: 
    kubectl delete all --all

## Demo 3: Container lifecycle handlers
    
    # preStart and postStop
    # we write into a file and before stop we write to stdout
    cat lifecycle-pod.yaml
    kubectl apply -f lifecycle-pod.yaml
    kubectl get pods -o wide
    # check the file that is created at preStart
    kubectl exec -it lifecycle-demo -- /bin/bash
    cat /usr/share/message
    
    # clean up in test namespace: 
    # before deleting the pod with preStop we gracefully shutdown nginx
    kubectl delete all --all

## Demo 4: Volumes 

    # volume - hostPath 
    # very important to bind the pod to the node where the path (and data) exists
    cat hostPath-volume-pod.yaml
    kubectl apply -f hostPath-volume-pod.yaml
    # check where the pod is scheduled
    kubectl get pods -o wide
    # shouldn't work since we do not have index.html in /tmp/html (even the folder does not exist)
    curl <POD-IP>
    # on the node where the pod runs 
    sudo -i 
    echo "hello" > /tmp/html/index.html
    # check again
    curl <POD-IP>
  
    # clean up in test namespace: 
    kubectl delete all --all
    
## Demo 5: Config - Command
    
    # config - command and arg
    # maybe a better example is from #DEMO 3 the stress container that is set to use CPUs
    cat config-command-pod.yaml
    kubectl apply -f config-command-pod.yaml
    kubectl logs config-command

    # clean up in test namespace: 
    kubectl delete all --all

## Demo 6: Config - Environment Variable
    
    # simple environment variable
    # we can pass environment variables as well that can be used by the application
    cat config-environment-variable-pod.yaml
    kubectl apply -f config-environment-variable-pod.yaml
    kubectl get pods -o wide
    kubectl exec -it envar-demo -- /bin/bash
    printenv

    # clean up in test namespace: 
    kubectl delete all --all
    
## Demo 7: Config - ConfigMap
    
    # configMap into the file system (but could be written into an environment variable as well)
    cat config-configMap-volume-pod.yaml      
    
    #create configMap and pod:
    kubectl apply -f config-configMap-volume-pod.yaml
    kubectl get pods -o wide
    kubectl get configmaps -o wide
    # notice the assigned configMap in pod description
    kubectl describe pod configmap-volume-pod
    kubectl describe configmaps app-config
    kubectl exec configmap-volume-pod -- cat /etc/config/game
      
    # config - Secret into environment variable (but could be written into a file as well)
    cat config-environment-variable-secret-pod.yaml
    
    # create secret (if created declaratively we need to give the encoded forms in the file)
    kubectl create secret generic mysql-pass --from-literal=password=123456 --dry-run=client -o yaml > config-environment-variable-secret.yaml
    cat config-environment-variable-secret.yaml
    
    kubectl apply -f config-environment-variable-secret.yaml
    kubectl apply -f config-environment-variable-secret-pod.yaml
    
    kubectl describe pod mysql
    kubectl describe secrets mysql-pass
    kubectl exec mysql -- printenv | grep -i pass
    # why secret is not secure? (Not encrypted only encoded)
    kubectl get secrets mysql-pass -o yaml
    # lets decode the encoded secret
    echo "MTIzNDU2" | base64 -d

    # clean up in test namespace: 
    kubectl delete all --all
        
## Demo 8: Security context
  
    # no capability
    cat security-context-pod.yaml
    # NET_ADMIN capability 
    cat security-context-2-pod.yaml

    kubectl apply -f security-context-pod.yaml
    kubectl apply -f security-context-2-pod.yaml
    
    kubectl get pods -o wide
    
    # check network interfaces
    kubectl exec security-context -- ip a
    kubectl exec security-context-2 -- ip a

    # check services
    curl <POD-IP>:8080
    curl <POD-IP>:8080
    
    # try to delete eth0
    # should not work for pod 1
    kubectl exec -it security-context -- ip addr del <POD-IP>/24 dev eth0
    kubectl exec -it security-context -- ip a
    curl <POD-IP>:8080
    
    # should work for pod 2 
    kubectl exec -it security-context-2 -- ip addr del <POD-IP>/24 dev eth0
    kubectl exec -it security-context-2 -- ip a
    curl <POD-IP>:8080

    # clean up in test namespace: 
    kubectl delete all --all
    
## Demo 9: ResourceQuota

    # set quota for CPU, memory and for number of pods
    cat resource-quota.yaml
    kubectl apply -f resource-quota.yaml
    kubectl get resourcequotas
    
    # initially we have enough resources to create all pods (that is currently one)
    cat quota-filler-deployment.yaml
    kubectl apply -f quota-filler-deployment.yaml
    kubectl get pods -o wide

    # check how we start to fill the quota
    kubectl get resourcequotas

    # edit deployment yaml to have 3 replicas
    nano quota-filler-deployment.yaml
    kubectl apply -f quota-filler-deployment.yaml
    # check that not all three pods can be started
    kubectl get pods -o wide
    kubectl describe deployments quota-filler
    kubectl describe replicasets quota-filler-<TAB>

    # clean up in test namespace: 
    kubectl delete all --all
    # need to be delete separately
    kubectl delete resourcequota <TAB>
  
## Demo 10: LimitRange

    # set default CPU limits to 300m and 200 < CPU < 800
    cat limit-range.yaml
    kubectl apply -f limit-range.yaml
    kubectl describe limitranges limit-range-test
    
    # no limit specification
    cat limitless-pod.yaml
    kubectl apply -f limitless-pod.yaml
    
    # check that its get the CPU settings
    kubectl get pod nginx -o yaml 
    
    kubectl delete -f limitless-pod.yaml
    
    # add resources request cpu 100m
    nano limitless-pod.yaml
    
    # notice that 100m is less than what we set in the limit range so an error should happen
    kubectl apply -f limitless-pod.yaml

    # clean up in test namespace: 
    kubectl delete all --all
    # need to be delete separately
    kubectl delete limitranges <TAB>
   
## Demo 11: Taints

    # add taint to a node
    kubectl taint node <NODE-NAME> leavemealone=please:NoSchedule
    kubectl describe nodes | grep Taint
    
    # this deployment should be scheduled
    # however, it is not guaranteed that will be scheduler on the tainted node
    cat tolerant-deployment.yaml
    kubectl apply -f tolerant-deployment.yaml
    
    kubectl get pods -o wide
   
    # scale deployment to 3
    nano tolerant-deployment.yaml
    kubectl apply -f tolerant-deployment.yaml
    
    # check that pods are not necesserily on the same node
    kubectl get pods -o wide

    # clean up in test namespace: 
    kubectl delete all --all
    kubectl taint node <NODE-NAME> leavemealone-
    kubectl describe nodes | grep Taint
      
## Demo 12: Node selectors

    # the pod only start on node(s) for which the selector is true
    # so it won't be scheduled since no node has the defined label
    cat selective-deployment.yaml
    kubectl apply -f selective-deployment.yaml
    kubectl get pods -o wide
    kubectl describe pod selective-app-<TAB>

    # add label to node
    kubectl label nodes <NODE-NAME> diskType=ssd
    kubectl get nodes --show-labels
    
    kubectl get pods -o wide
    
    # remove node label
    kubectl label nodes <NODE-NAME> diskType-
    kubectl get nodes --show-labels
    
    # K8s doesn't kill running Pods (usually)
    kubectl get pods -o wide
    
    # so we kill the Pod
    kubectl delete pod selective-app-<TAB>

    # notice pending state
    kubectl get pods -o wide

    # clean up in test namespace: 
    kubectl delete all --all

## Demo 13: Anti-affinity
    
    cat anti-affinity-deployment.yaml
    kubectl apply -f anti-affinity-deployment.yaml
    
    # check where the pods are placed
    kubectl get pods -o wide
    
    # scale deployment to 2
    nano anti-affinity-deployment.yaml
    kubectl apply -f anti-affinity-deployment.yaml
    
    # check where the pods are placed
    kubectl get pods -o wide
    
    # scale deployment to 4
    nano anti-affinity-deployment.yaml
    kubectl apply -f anti-affinity-deployment.yaml
    
    # notice that one pod stuck in pending state
    kubectl get pods -o wide
    
    kubectl describe pod anti-affinity-app-<TAB>
    
    # clean up in test namespace: 
    kubectl delete all --all
    
## Demo 14: Troubleshoot

    # stop kubelet on a node
    kubectl get nodes
    sudo systemctl status kubelet.service
    sudo systemctl stop kubelet.service
    # should see that the node has NotReady state
    # however, pod on the node won't be rescheduled immediately (5 min by default)
    kubectl get nodes
    
    # bring kubectl back
    sudo systemctl start kubelet.service
    sudo systemctl status kubelet.service
    kubectl get nodes

Problem 1:

    # cat bad-1-deployment.yaml
    kubectl apply -f bad-1-deployment.yaml
    kubectl get deployment
    kubectl describe pods nginx
    # fix the yaml
    nano bad-1-deployment.yaml
    # re-apply 
    kubectl apply -f bad-1-deployment.yaml
    kubectl get deployment
    kubectl delete -f bad-1-deployment.yaml
    
Problem 2:

    kubectl apply -f bad-2-deployment.yaml
    kubectl get deployment
    kubectl get pods -o wide
    kubectl describe pods busy-<TAB>
    kubectl logs busy-<TAB>
    # fix the yaml
    nano bad-2-deployment.yaml
    # re-apply 
    kubectl apply -f bad-2-deployment.yaml
    kubectl get deployment
    kubectl delete -f bad-2-deployment.yaml
    
Problem 3:

    kubectl apply -f bad-3-deployment.yaml
    kubectl get service
    kubectl get deployments
    kubectl get pods -o wide
    curl <SERVICE-IP>
    curl <POD-IP>
    kubectl describe service normal-service
    kubectl get endpoints normal-service
    # fix the yaml
    nano bad-3-deployment.yaml
    # re-apply 
    kubectl apply -f bad-3-deployment.yaml
    kubectl get endpoints normal-service
    curl <SERVICE-IP>
    
    # clean up in test namespace: 
    kubectl delete all --all
