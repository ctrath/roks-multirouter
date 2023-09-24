# roks-multirouter
IBM Redhat Openshift Kubernetes Service - multiple router example


#### Create a load balancer service
I'm creating a load balancer service in order to generate a public entrypoint into
my router.  The load balancer service will create a public VPC ALB.

Note that ALB creating isn't completely necessary.  For instance, if you plan to
connect to the router over the private network, you may wish to create a private LoadBalancer
service or a Nodeport service instead.

Apply load balancer -
`kubectl apply -f app-ingress-controller-lb-service.yaml`

Check load balancer status-
`kubectl describe svc -n openshift-ingress approuter`

You want to make sure the load balancer is started.  A healthy, started ALB will have
a status that looks similar to this:
```
Events:
  Type    Reason                Age    From                Message
  ----    ------                ----   ----                -------
  Normal  EnsuringLoadBalancer  5m22s  service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   4m8s   service-controller  Ensured load balancer
```
Note: `EnsuredLoadBalancer` denotes that the ALB is ready


#### Create a domain and certificate for the load balancer
I'm creating a domain and certificate for my secondary router as I don't have one
already to use.  If you already have a domain and certificate, you can skip this section
and store the certificate in a secret in the cluster that the secondary router can access
for TLS enablement.

Get the name of your cluster:
```
>ibmcloud ks cluster ls
OK
Name                          ID                     State    Created       Workers   Location   Version                  Resource Group Name   Provider
mycluster-au-syd-1-bx2.4x16   ck5jlt1s01b1sr0pdrd0   normal   1 day ago     4         Sydney     4.13.11_1540_openshift   default               vpc-gen2
```

Get the domain name of your ALB that was created in the previous section:
```
>kubectl get svc -n openshift-ingress
NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP                          PORT(S)                      AGE
approuter                 LoadBalancer   172.21.40.13    3f6b8d29-au-syd.lb.appdomain.cloud   80:31898/TCP,443:31987/TCP   18m
router-default            LoadBalancer   172.21.175.57   c20f5384-au-syd.lb.appdomain.cloud   80:32605/TCP,443:32761/TCP   47h
router-internal-default   ClusterIP      172.21.17.208   <none>                               80/TCP,443/TCP,1936/TCP      47h
```

Create a domain name and certificate for the load balancer service:

```
>ibmcloud ks nlb-dns create vpc-gen2 -c mycluster-au-syd-1-bx2.4x16 --secret-namespace openshift-ingress --type public --lb-host 3f6b8d29-au-syd.lb.appdomain.cloud
```

Check the status of the domain and certificate creation:
```
❯ ibmcloud ks nlb-dns ls -c mycluster-au-syd-1-bx2.4x16
OK
Subdomain                                                                                           Target(s)                            SSL Cert Status   SSL Cert Secret Name                                              Secret Namespace    Status
mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0000.au-syd.containers.appdomain.cloud   c20f5384-au-syd.lb.appdomain.cloud   created           mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0000   openshift-ingress   OK
mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud   3f6b8d29-au-syd.lb.appdomain.cloud   created           mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001   openshift-ingress   OK
```

Note: match the Target domain name, in this case `3f6b8d29-au-syd.lb.appdomain.cloud`, with the load balancer service `EXTERNAL-IP` that was listed earlier.  When the domain and certificate are created, the Status indicate `OK`


Check to make sure the certificate was generated:
```
❯ kubectl get secret -n openshift-ingress
NAME                                                              TYPE                                  DATA   AGE
builder-dockercfg-h94wn                                           kubernetes.io/dockercfg               1      47h
builder-token-747bx                                               kubernetes.io/service-account-token   4      47h
default-dockercfg-hh2lw                                           kubernetes.io/dockercfg               1      47h
default-token-cmd2f                                               kubernetes.io/service-account-token   4      47h
deployer-dockercfg-kkqnb                                          kubernetes.io/dockercfg               1      47h
deployer-token-22wh9                                              kubernetes.io/service-account-token   4      47h
mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0000   kubernetes.io/tls                     2      47h
mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001   kubernetes.io/tls                     2      4m2s
router-dockercfg-bqzqs                                            kubernetes.io/dockercfg               1      47h
router-metrics-certs-default                                      kubernetes.io/tls                     2      47h
router-stats-default                                              Opaque                                2      47h
router-token-ppd6f                                                kubernetes.io/service-account-token   4      47h
```

Note: the secret `mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001` contains the certificate that we're interested in and can be mapped to the domain name that was listed in the previous step.

#### Modify the ingress controller yaml in this repo with your domain and certificate (app-ingress-controller.yaml)
```
apiVersion: v1
items:
- apiVersion: operator.openshift.io/v1
  kind: IngressController
  metadata:
    name: approuter
    namespace: openshift-ingress-operator
  spec:
    defaultCertificate:
      name: mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001
    domain: mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud
    namespaceSelector:
      matchLabels:
        router: app
    routeAdmission:
      namespaceOwnership: InterNamespaceAllowed
```

You will need to modify following fields accordingly:
 - spec.defaultCertificate.name
 - spec.domain
 - spec.namespaceSelector.matchLabels

Save and apply the yaml:
`kubectl apply -f app-ingress-controller.yaml`

Validate the ingress controller is deployed:
```
❯ kubectl get ingresscontroller -n openshift-ingress-operator
NAME        AGE
approuter   26s
default     2d1h
```

Note: as you can see in the output from the command above, the new ingress controller `approuter` is deployed


#### Application deploy and ingress resource definition
In the namespace which you want to serve up application from the new ingress controller,
you need to add an annotation on each namespace that you want it to serve:

`kubectl label namespace default router=app`

Note: the label is the same label as specified in the namespace selector in the IngressController definition above

To deploy a sample app run:
```
❯ kubectl apply -f echoserverdeploy.yaml
deployment.apps/echo-server created
❯ kubectl apply -f echoserverclusterservice.yaml
service/echoserverservice created
❯ kubectl apply -f echoingressresource.yaml
ingress.networking.k8s.io/echoserveringressresource created
```

Validate the ingress works:
```
❯ curl http://echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud


Hostname: echo-server-76797d9f6-4sngp

Pod Information:
	node name:	10.245.0.8
	pod name:	echo-server-76797d9f6-4sngp
	pod namespace:	default
	pod IP:	172.17.67.176

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=172.17.70.158
	method=GET
	real path=/
	query=
	request_version=1.1
	request_scheme=http
	request_uri=http://echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud:8080/

Request Headers:
	accept=*/*
	forwarded=for=10.245.0.9;host=echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud;proto=http
	host=echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud
	user-agent=curl/8.1.2
	x-forwarded-for=10.245.0.9
	x-forwarded-host=echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud
	x-forwarded-port=80
	x-forwarded-proto=http

Request Body:
	-no body in request-
```


#### Restricing default ingress from accessing application namespaces

We're not done yet.  The default ingress controller still has access to the ingress resource in all namespaces,
including the default namespace.  You can test this by running a curl against the default ingress host with
the hostname header using the host name of the application controller.  For example:
```
❯ curl --header "Host: echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud" c20f5384-au-syd.lb.appdomain.cloud


Hostname: echo-server-76797d9f6-4sngp

Pod Information:
	node name:	10.245.0.8
	pod name:	echo-server-76797d9f6-4sngp
	pod namespace:	default
	pod IP:	172.17.67.176

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=172.17.67.145
	method=GET
	real path=/
	query=
	request_version=1.1
	request_scheme=http
	request_uri=http://echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud:8080/

Request Headers:
	accept=*/*
	forwarded=for=10.245.0.9;host=echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud;proto=http
	host=echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud
	user-agent=curl/8.1.2
	x-forwarded-for=10.245.0.9
	x-forwarded-host=echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud
	x-forwarded-port=80
	x-forwarded-proto=http

Request Body:
	-no body in request-
```

So, in order to prevent the default ingress controller from access the default namespace and only serve up the console, we'll add a
namespace selector the default ingress controller and also annotate the openshift-console namespace.
`kubectl edit ingresscontroller -n openshift-ingress-operator default`

Add the namespaceSelector to look something like this and save the ingressController resource:
```
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"operator.openshift.io/v1","kind":"IngressController","metadata":{"annotations":{},"name":"default","namespace":"openshift-ingress-operator"},"spec":{"endpointPublishingStrategy":{"loadBalancer":{"scope":"External"},"type":"LoadBalancerService"},"nodePlacement":{"tolerations":[{"key":"dedicated","value":"edge"}]}},"status":{}}
    razee.io/build-url: https://travis.ibm.com/alchemy-containers/armada-ingress-secret-mgr/builds/6174279
    razee.io/source-url: https://github.ibm.com/alchemy-containers/armada-ingress-secret-mgr/commit/a898ae7846e7a8e056c58832864f5283f1950a5c
  creationTimestamp: "2023-09-20T18:41:03Z"
  finalizers:
  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
  generation: 3
  name: default
  namespace: openshift-ingress-operator
  resourceVersion: "1628598"
  uid: 3dd316da-ed84-4506-b846-b0575fb1badb
spec:
  clientTLS:
    clientCA:
      name: ""
    clientCertificatePolicy: ""
  defaultCertificate:
    name: mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0000
  endpointPublishingStrategy:
    loadBalancer:
      dnsManagementPolicy: Managed
      scope: External
    type: LoadBalancerService
  httpCompression: {}
  httpEmptyRequestsPolicy: Respond
  httpErrorCodePages:
    name: ""
  namespaceSelector:
    matchLabels:
      router: default
  nodePlacement:
    tolerations:
    - key: dedicated
      value: edge
  tuningOptions:
    reloadInterval: 0s
  unsupportedConfigOverrides: null
status:
  availableReplicas: 1
```

Now, annotate the openshift-console namespace so the default ingress controller has access:
`❯ kubectl label namespace openshift-console router=default`

Now, if you rerun the curl that was ran earlier, attempting to access the openshift console using the application ingress
controller domain, you will no longer have access, though you will still have access using the default ingress controller domain:
`❯ curl --header "Host: echo.mycluster-au-syd-1-bx2-4x-aa26ccd186043d00655b29246b83475c-0001.au-syd.containers.appdomain.cloud" c20f5384-au-syd.lb.appdomain.cloud`

