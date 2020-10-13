# Networking in Kubernetes

## Basics of CNI

The Container Networking Interface ([CNI](https://github.com/containernetworking/cni)) is really simple interface to attach a network interface Kubernetes pods.
Kubelet requires a descriptor file in `json` format to be present at `/etc/cni/net.d`:
```
root@k8s-master:~# ls -lh /etc/cni/net.d
total 4.0K
-rw-r--r-- 1 root root 292 Sep 24 08:20 10-flannel.conflist
```

An example content of such `10-flannel.conflist` file looks like this:
```
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

The `type` field will tell Kubelet which binary should it execute after every pod creation/deletion.
The binaries are usually located under `/opt/cni/bin`:
```
-rwxr-xr-x 1 root root 4.0M May 13 19:50 bandwidth
-rwxr-xr-x 1 root root 4.5M May 13 19:50 bridge
-rwxr-xr-x 1 root root  12M May 13 19:50 dhcp
-rwxr-xr-x 1 root root 5.7M May 13 19:50 firewall
-rwxr-xr-x 1 root root 3.0M May 13 19:50 flannel
-rwxr-xr-x 1 root root 4.0M May 13 19:50 host-device
-rwxr-xr-x 1 root root 3.5M May 13 19:50 host-local
-rwxr-xr-x 1 root root 4.2M May 13 19:50 ipvlan
-rwxr-xr-x 1 root root 3.1M May 13 19:50 loopback
-rwxr-xr-x 1 root root 4.2M May 13 19:50 macvlan
-rwxr-xr-x 1 root root 3.8M May 13 19:50 portmap
-rwxr-xr-x 1 root root 4.4M May 13 19:50 ptp
-rwxr-xr-x 1 root root 3.3M May 13 19:50 sbr
-rwxr-xr-x 1 root root 2.8M May 13 19:50 static
-rwxr-xr-x 1 root root 3.3M May 13 19:50 tuning
-rwxr-xr-x 1 root root 4.2M May 13 19:50 vlan
```

Creating an network interface for a new pod usually means to create virtual Ethernet interface (veth) pair,
attach one of them to a Linux Bride (`cni0` based on this descriptor), while put the other to the pod's namesapce
and configure a unique IP.

You can actually manually execute commands in the pod's networking namespace using the following example:
```
# get the ID of a "pause" container, e.g.
$ docker ps | grep pause | head -1
b3d9bb330e9b        k8s.gcr.io/pause:3.2   "/pause"                 28 hours ago        Up 28 hours                             k8s_POD_coredns-f9fd979d6-7frcb_kube-system_9f481df9-1d2a-40d0-ba07-81b96e4af2d4_7
# inspect the container and search for netns for the network namespace reference file
$ docker inspect $(docker ps | grep pause | head -1 | cut -d' ' -f1) | grep SandboxKey
            "SandboxKey": "/var/run/docker/netns/5946702ca2e8",
# you have to create a symlink to this file at /var/run/netns
$ mkdir -p /var/run/netns
$ MYNETNS=$(docker inspect $(docker ps | grep pause | head -1 | cut -d' ' -f1) | grep SandboxKey | cut -d'"' -f4)
$ ln -s $MYNETNS /var/run/netns/mynetns
$ ip netns exec mynetns ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1410 qdisc noqueue state UP group default 
    link/ether 96:3e:73:35:ff:cc brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.0.12/24 brd 10.244.0.255 scope global eth0
       valid_lft forever preferred_lft forever
# you should see the same IP adderss for eth0 with this command:
$ kubectl get pod -n kube-system coredns-f9fd979d6-7frcb -o jsonpath='{.status.podIP}'
10.244.0.12
```

## Using Services in Kubernetes

```
# create webserver deployment with 3 replicas
cat webserver-deployment.yaml
kubectl apply -f webserver-deployment.yaml
kubectl get pods -o wide

# delete one pod to see that the new pod will have new IP
kubectl delete $(kubectl get pods -o name | head -1)
kubectl get pods -o wide

# so let's create a service for the pods in this deployment
cat webserver-service.yaml
kubectl apply -f webserver-service.yaml
kubectl get service

# now lets call the service
curl $(kubectl get service webserver-service -o jsonpath='{.spec.clusterIP}')

# do that in an infinite loop to see the load balancing
while true; do curl $(kubectl get service webserver-service -o jsonpath='{.spec.clusterIP}'); sleep 1; done

# inside the pods the DNS is also available
kubectl exec -it $(kubectl get pods -o name | head -1) -- nslookup webserver-service
kubectl exec -it $(kubectl get pods -o name | head -1) -- nslookup webserver-service.test.svc.cluster.local

# it work since CoreDNS also has a service and that IP is mounted in the pod's resolv.conf file
kubectl get services -n kube-system
kubectl exec -it $(kubectl get pods -o name | head -1) -- cat /etc/resolv.conf

# you can also see the endpoints that participate in the service
kubectl get endpoints

# using NodePort type service
cat nodeport-service.yaml
kubectl apply -f nodeport-service.yaml
kubectl get services

# check the port with curl using localhost
curl localhost:$(kubectl get service nodeport-service -o jsonpath="{.spec.ports[0].nodePort}")

# alternatively, you can try any other nodePort
kubectl get nodes -o wide
curl $(kubectl get nodes -o jsonpath="{.items[1].status.addresses[0].address}"):$(kubectl get service nodeport-service -o jsonpath="{.spec.ports[0].nodePort}")

# using LoadBalancer type service
cat loadbalancer-service.yaml
kubectl apply -f loadbalancer-service.yaml
kubectl get services
kubectl describe service loadbalancer-service
# this won't work, so repeat it on on a managed cloud environment, which will give you a public IP

# headless services don't have ClusterIP, rather they create DNS names for all pod IPs
cat headless-service.yaml 
kubectl apply -f headless-service.yaml 
kubectl get services

# there is no ClusterIP to access the service and the Endpoints are the same
kubectl get endpoints

# so lets check the DNS in a pod
kubectl exec -it $(kubectl get pods -o name | head -1) -- nslookup headless-service.test.svc.cluster.local

# internal services can resolve to external services using the type=ExternalName
cat externalname.yaml
kubectl apply -f externalname.yaml
kubectl get services

# check the DNS inside a pod
kubectl exec -it $(kubectl get pods -o name | head -1) -- ping externalname-service

# can also map and internal service and ClusterIP to an external IP using a (headless) service no selector, and adding a static endpoint to the service
cat external-static-ip.yaml
kubectl apply -f external-static-ip.yaml 
kubectl get services
kubectl get endpoints

# now add a static endpoint with the same name as the service
cat static-endpoint.yaml
kubectl apply -f static-endpoint.yaml
kubectl get endpoints 

# try ping the service from inside a container (ping will only work in case of headless service, you can never ping a ClusterIP)
kubectl exec -it $(kubectl get pods -o name | head -1) -- ping external-ip
```
## Using an ingress:

Install an Nginx ingress controller.
Official docs at: https://kubernetes.github.io/ingress-nginx/deploy/
```
# install the ingress controller by
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.35.0/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx

# you can see that the service uses NodePort by default
kubectl get services -n ingress-nginx

# you can can change that to LoadBalancer with the following patch command
kubectl patch -n ingress-nginx service ingress-nginx-controller -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get services -n ingress-nginx

# you can check if the ingress controller runs by:
curl <VM IP>:<nginx node port>
```


Using the Ingress object:
```
# install an example ingress
cat ingress-webserver.yaml
kubectl apply -f ingress-webserver.yaml
kubectl get ingress

# try to curl the ingress, you will get 404 error
curl <VM IP>:<nginx node port>

# you have to specify the ingress class to work
cat ingress-with-class.yaml
kubectl apply -f ingress-with-class.yaml
kubectl get ingress
curl <VM IP>:<nginx node port>

# delete the ingress for the next test
kubectl delete -f ingress-with-class.yaml

# create and ingress that will be routed with a hotname
cat ingress-with-hostname.yaml
kubectl apply -f ingress-with-hostname.yaml
kubectl get ingress

# normal curl will get a 404 error
curl <VM IP>:<nginx node port>

# you can specify the hostname in the HTTP header, and the query should execute
curl -h 'Host: hwsw-k8s.leannet.eu' <VM IP>:<nginx node port>
```

SSL/TLS support in Ingress:
```
# Ingress controllers usually have a built-in self signed cert, try curl the https port
kubectl get services -n ingress-nginx
curl -h 'Host: hwsw-k8s.leannet.eu' https://<VM IP>:<HTTPS node port>

# since it is self-signed, curl will block it unless insecure mode is on
curl -k -h 'Host: hwsw-k8s.leannet.eu' https://<VM IP>:<HTTPS node port>

# you can enable automatic redirect to SSL port with 'nginx.ingress.kubernetes.io/ssl-redirect: "true"' annotation

# you can also use your own cert, you have to create a secret as follows:
    apiVersion: v1
    kind: Secret
    metadata:
      name: my-tls
      namespace: default
    data:
      tls.crt: base64 encoded cert
      tls.key: base64 encoded key
    type: kubernetes.io/tls

# and include the secret name in your ingress object
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: webserver-ingress  
    spec:
      tls:
      - hosts:
          - hwsw-k8s.leannet.eu
        secretName: my-tls
      ingressClassName: nginx
      rules:
      - host: hwsw-k8s.leannet.eu
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webserver-service
                port:
                  number: 80
```

Use a valid certificate with Cert-Manager and Let's Encrypt.
```
# first, create two deployments with two services
cat blue-green.yaml
kubectl apply -f blue-green.yaml
kubectl get pods

# create the ingress
cat blue-green-ingress.yaml
kubectl apply -f blue-green-ingress.yaml
kubectl get ingress

# you should see and error of you try to access the https port, but with insecure access it will work:
curl https://blue.leannet.hu
curl -k https://blue.leannet.hu

# install Cert-Manager
# for Kubernetes version 1.16+
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml

# for Kubernetes version <1.16
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager-legacy.yamlkubectl get pods -n cert-manager

# create a certificate issuer
cat cert-issuer.yaml
kubectl apply -f cert-issuer.yaml
kubectl get issuers.cert-manager.io

# now modify the ingress to annotate with the issuer and and add the TLS secret that will be automatically created by Cert-Manager:
cat ingress-with-cert-manager.yaml
kubectl apply -f ingress-with-cert-manager.yaml
kubectl get ingress
kubectl get certificaterequests.cert-manager.io
kubectl get certificates.cert-manager.io
kubectl get secret

# if everything is in ready state, try the curl again without the insecure option
curl https://blue.leannet.hu
```
