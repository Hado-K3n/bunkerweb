# Install Bunkerweb on your managed K8S cluster

## Preview of the final infrastructure 
![Step 7](./images/schema-LB-ingress.png
)

## STEP 1 - Setup your managed kubernetes cluster on OVH
![Step 1](./images/Managed-cluster-1.png)

![Step 2](./images/Managed-cluster-2.png)

![Step 3](./images/Managed-cluster-3.png)

![Step 4](./images/Managed-cluster-4.png)

![Step 5](./images/Managed-cluster-5.png)

![Step 6](./images/Managed-cluster-6.png)

![Step 7](./images/Managed-cluster-7.png)
##  STEP 2 -  Bunkerweb installation 

Bunkerweb can easily be installed from the yaml file you would find on [Bunkerity's Github](https://github.com/bunkerity/bunkerweb/tree/master/misc/integrations) under the same prefix __"K8S.*.yaml"__.

Documentation for BunkerWeb is regularly updated to fit the latest version, you can find it [here](https://docs.bunkerweb.io/).

![Bunkerweb K8S repository](./images/Github.png )

Once your managed kubernetes cluster is in place, you need to personnalize your downloaded yaml file to fit your specifications.

###  STEP 2.1 - Specify the DNS for your cluster

```
          env:
            - name: KUBERNETES_MODE
              value: "yes"
            # replace with your DNS resolvers
            # e.g. : kube-dns.kube-system.c.cluster.local
            - name: DNS_RESOLVERS
              value: "kube-dns.kube-system.svc.cluster.local"
```
###  STEP 2.2 - Specify the environment variables needed for the proxy setup

```
            - name: USE_PROXY_PROTOCOL
              value: "yes"
            - name: USE_REAL_IP
              value: "yes"
            - name: REAL_IP_FROM
              value: "10.0.0.0/8"
            - name: REAL_IP_HEADER
              value: "proxy_protocol"
```
###  STEP 2.3 -  Change the default values 

You should change the default database password. For example "changeme" to "testor":


e.g.:
```
            - name: "POSTGRES_PASSWORD"
              value: "testor"
```
Don't forget to modify it as well in your database URI for the scheduler, the controller and the UI deployments. 

From
```
            - name: "DATABASE_URI"
              value: "postgresql://bunkerweb:changeme@svc-bunkerweb-db:5432/db"
```
to
```
            - name: "DATABASE_URI"
              value: "postgresql://bunkerweb:testor@svc-bunkerweb-db:5432/db"
```

###  STEP 2.4 -  Change the default UI values

From
```
      containers:
        - name: bunkerweb-ui
          image: bunkerity/bunkerweb-ui:1.5.0
          imagePullPolicy: Always
          env:
            - name: ADMIN_USERNAME
              value: "changeme"
            - name: "ADMIN_PASSWORD"
              value: "changeme"
            - name: "ABSOLUTE_URI"
              value: "http://www.example.com/changeme/"
            - name: KUBERNETES_MODE
              value: "YES"
            - name: "DATABASE_URI"
              value: "postgresql://bunkerweb:changeme@svc-bunkerweb-db:5432/db"
```
to 

```
      containers:
        - name: bunkerweb-ui
          image: bunkerity/bunkerweb-ui:1.5.0
          imagePullPolicy: Always
          env:
            - name: ADMIN_USERNAME
              value: "admin"
            - name: "ADMIN_PASSWORD"
              value: "4ltestorO!"
            - name: "ABSOLUTE_URI"
              value: "http://ip-152-228-251-184.gra.lb.ovh.net/admin/" # specify your LB URL
            - name: KUBERNETES_MODE
              value: "YES"
            - name: "DATABASE_URI"
              value: "postgresql://bunkerweb:testor@svc-bunkerweb-db:5432/db"
```
It's important to note that your URL needs to end with a "/"

Also important to note, the admin password must contain at least 8 characters, including at least 1 uppercase letter, 1 lowercase letter, 1 number and 1 special character (#@?!$%^&*-).

## STEP 3 - Install an application for testing purposes
We can install an Hello-world deployment.

You can find OVH documentation on the LoadBalancer [here](https://help.ovhcloud.com/csm/en-public-cloud-kubernetes-using-lb?id=kb_article_view&sysparm_article=KB0050019).
```
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: hello-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: ovhplatform/hello:1.1
        ports:
        - containerPort: 80
```


## STEP 4 -  Change the default ingress values

From
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    bunkerweb.io/www.example.com_USE_UI: "yes"
    bunkerweb.io/www.example.com_REVERSE_PROXY_HEADERS_1: "X-Script-Name /changeme"
    bunkerweb.io/www.example.com_INTERCEPTED_ERROR_CODES: "400 404 405 413 429 500 501 502 503 504"
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /changeme/
            pathType: Prefix
            backend:
              service:
                name: svc-bunkerweb-ui
                port:
                  number: 7000
```

to

``` 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    bunkerweb.io/ip-152-228-251-184.gra.lb.ovh.net_USE_UI: "yes"
    bunkerweb.io/ip-152-228-251-184.gra.lb.ovh.net_REVERSE_PROXY_HEADERS_1: "X-Script-Name /admin"
    bunkerweb.io/ip-152-228-251-184.gra.lb.ovh.net_INTERCEPTED_ERROR_CODES: "400 404 405 413 429 500 501 502 503 504"
spec:
  rules:
    - host: ip-152-228-251-184.gra.lb.ovh.net
      http:
        paths:
          - path: /admin/
            pathType: Prefix
            backend:
              service:
                name: svc-bunkerweb-ui
                port:
                  number: 7000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-world
                port:
                  number: 8080
```

## STEP 5 - LoadBalancer with OVH

### STEP 5.1 - Create your service type LoadBalancer

```
apiVersion: v1
kind: Service
metadata:
  name: lb-bw
  annotations:
    service.beta.kubernetes.io/ovh-loadbalancer-proxy-protocol: "v2"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: bunkerweb
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
    - port: 443
      targetPort: 8443
      protocol: TCP
      name: https
```
### STEP 5.2 - Find out your LoadBalancer IP
 Type  ```kubectl get service``` and catch your LoadBalancer IP  in the EXTERNAL-IP column. 
```
➜  K8S git:(master) ✗ kubectl get service        
NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP                         PORT(S)                      AGE
hello-world           ClusterIP      10.3.58.37     <none>                              8080/TCP                     164m
kubernetes            ClusterIP      10.3.0.1       <none>                              443/TCP                      24h
lb-bw                 LoadBalancer   10.3.125.204   ip-152-228-251-184.gra.lb.ovh.net   80:31880/TCP,443:30468/TCP   3h14m
svc-bunkerweb         ClusterIP      None           <none>                              <none>                       164m
svc-bunkerweb-db      ClusterIP      10.3.167.212   <none>                              5432/TCP                     164m
svc-bunkerweb-redis   ClusterIP      10.3.27.181    <none>                              6379/TCP                     164m
svc-bunkerweb-ui      ClusterIP      10.3.193.146   <none>                              7000/TCP                     164m

```

## STEP 6 - Try to reach your instances 

- BunkerWeb should be reachable through this path: http://ip-152-228-251-184.gra.lb.ovh.net/admin/


![BunkerWeb UI interface](./images/BW-cap.png)

- OVH Hello-World application should be reachable through this path: http://ip-152-228-251-184.gra.lb.ovh.net


![OVH Hello-world ](./images/OVH-cap.png)

As specified in the Ingress you should reach your destination.

**Your application is now Bunkerized and protected from malevolent outsiders !**

