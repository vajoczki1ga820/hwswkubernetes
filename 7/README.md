## Demo 1: Take a look at K8s public key infrascturcture (PKI)
   
    # notice ca.crt/key, <CONMPONENT-NAME>.crt/key and *.-client.crt/key files
    ll /etc/kubernetes/pki/
    ll /etc/kubernetes/pki/etcd

    # apiserver related certs are: a server side and two client side certificates 
    # client certs are set for both communications parties (apiserver - etcd, apiserver - kubelet)
    ps aux | grep apiserver | grep "apiserver-etcd-client"
    ps aux | grep apiserver | grep "apiserver-kubelet-client"
    ps aux | grep kubelet | grep "apiserver-kubelet-client"
    ps aux | grep kubelet | grep "apiserver"
    ps aux | grep etcd | grep "apiserver-etcd-client"
    ps aux | grep etcd | grep "apiserver"

    # take a look at a cert file:
    cat /etc/kubernetes/pki/apiserver.crt
    cat /etc/kubernetes/pki/apiserver.crt | openssl x509 -text
    
    # where is kubelet.crt? 
    # take a look at kubelet certificates
    ll /var/lib/kubelet/pki/
    cat /var/lib/kubelet/pki/kubelet.crt | openssl x509 -text
    
    # check the difference between existing cert files on a master and a worker node
    ll /etc/kubernetes/pki/
    ll /etc/kubernetes/pki/etcd
    ll /var/lib/kubelet/pki/
      
    # check apiserver pod Volumes: and Mounts: for certificates 
    kubectl -n kube-system describe pod kube-apiserver-<TAB>
      
    # certification expiry
    kubeadm alpha certs check-expiration
    
## Demo 2: Check an existing service account with roles and role bindings

    # look for sealed-secrets-controller
    kubectl -n kube-system get serviceaccounts
    kubectl -n kube-system get role sealed-secrets-key-admin
    
    # check one of its roles and roleBindings:
    kubectl -n kube-system get clusterrole secrets-unsealer
    kubectl -n kube-system get clusterrole secrets-unsealer -o yaml
    # better way to check it
    kubectl -n kube-system describe clusterrole secrets-unsealer
  
    kubectl -n kube-system get clusterrolebindings.rbac.authorization.k8s.io sealed-secrets-controller
    kubectl -n kube-system get clusterrolebindings.rbac.authorization.k8s.io sealed-secrets-controller -o yaml
    # better way to check it
    kubectl -n kube-system describe clusterrolebindings.rbac.authorization.k8s.io sealed-secrets-controller
    
    # check how the related pod gets its permission
    # notice token name
    kubectl -n kube-system describe serviceaccounts sealed-secrets-controller

    # notice token name copied into the pod -> check Volumes: and Mounts:
    kubectl -n kube-system describe pod sealed-secrets-controller-<TAB>
    
    # how k8s knows to attach this token?
    # check pod definition
    kubectl -n kube-system get pod sealed-secrets-controller-d7dd9db65-ws4wr -o yaml | grep -i serviceaccountname
    
## Demo 3: Accessing kubernetes via RESTful API from outside a Pod

    kubectl -n kube-system describe serviceaccounts sealed-secrets-controller
    kubectl -n kube-system describe secrets sealed-secrets-controller-token-<TAB>
    export MYBEARERTOKEN=<TOKEN-VALUE>
    # get server address and port
    kubectl config view | grep server
    export SERVER=<SERVER-VALUE>
    
    # list secrets
    curl -k \
    -H "Authorization: Bearer $MYBEARERTOKEN" \
    -H 'Accept: application/json' \
    $SERVER/api/v1/namespaces/kube-system/secrets/
    
## Demo 4: Create a "developer" user with limited access to the test namespace 

    # create the test namespace
    kubectl create namespace test
    
    # create the user credentials (K8s does not have API objects for this but OpenSSL certificates with key-value pairs can work well):
    # private key for your user
    openssl genrsa -out developer.key 2048
    # CSR (CN is for the username and O for the group)
    # only run this if openssl complains about .rnd file: openssl rand -out ./.rnd -hex 256
    openssl req -new -key developer.key -out developer.csr -subj "/CN=developer/O=leannet"
    # CRT by signing csr with cluster CA
    sudo openssl x509 -req -in developer.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out developer.crt -days 500
    # add to K8s config as credential
    # check the original config
    kubectl config view
    kubectl config set-credentials developer --client-certificate=./developer.crt  --client-key=./developer.key
    # set context
    kubectl config set-context developer-context --cluster=kubernetes --namespace=test --user=developer
    
    # get pods (won't work since at this point we only have a user but without permissions --> RBAC)
    kubectl --context=developer-context get pods
    
    # create roles and rolebindings
    cat developer-role.yaml
    kubectl create -f developer-role.yaml
    cat developer-role-binding.yaml
    kubectl create -f developer-role-binding.yaml
    
    # check pods again (shoul work now)
    kubectl --context=developer-context get pods
    
    kubectl --context=developer-context create deployment nginx --image nginx
    # should not work
    kubectl --context=developer-context create service clusterip nginx --tcp=5678:8080
    # should work
    kubectl create service clusterip nginx --tcp=5678:8080
    
    kubectl delete all --all
    
## Demo 5: Network policy

    # Only work if CNI plugin is not Flannel and even on a managed cluster (e.g. GKE) probably need to be enabled explicitly

    cat frontend-pod-and-service.yaml
    kubectl apply -f frontend-pod-and-service.yaml
    cat curl-pod.yaml
    kubectl apply -f curl-pod.yaml
    
    # should work
    kubectl exec -it busy -- curl nginx
    
    # create network policy to block the connection from busy
    cat block-budy-nginx-netwokr-policy.yaml
    kubectl apply -f block-budy-nginx-netwokr-policy.yaml
    
    # should not work
    kubectl exec -it busy -- curl nginx
    
## Demo 6: Sealed Secret

    # install 
    
    # client side (not need to be on a cluster node)
    wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.12.6/kubeseal-linux-amd64 -O kubeseal
    sudo install -m 755 kubeseal /usr/local/bin/kubeseal

    # cluster side
    kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.12.6/controller.yaml

    # check pod "sealed-secrets-controller"
    kubectl -n kube-system get pods -o wide

    # usage

    # create a json/yaml-encoded Secret somehow:
    # (note use of `--dry-run` - this is just a local file!)
    echo -n bar | kubectl create secret generic mysecret --dry-run=client --from-file=foo=/dev/stdin -o yaml >mysecret.yaml
    
    # check
    cat mysecret.yaml
    
    # crete sealed secret (still just a local file):
    kubeseal <mysecret.yaml >mysealedsecret.yaml
    
    # check 
    cat mysealedsecret.yaml
    
    # create the secret in K8s!
    kubectl create -f mysealedsecret.yaml
    
    # check as it is created as a normal secret
    kubectl get secrets
    kubectl describe secrets mysecret
    # take a look a "foo:" value
    kubectl get secrets mysecret -o yaml
    echo <FOO-VALUE> | base64 -d