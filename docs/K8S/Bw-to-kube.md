# Install Bunkerweb on your managed K8S cluster
## Setup your managed kubernetes cluster on OVH


## Bunkerweb installation 

Bunkerweb can easily be installed from the yaml file you would find on [Bunkerity's Github](https://github.com/bunkerity/bunkerweb/tree/master/misc/integrations) under the same prefix __"K8S.*.yaml"__.

![Bunkerweb K8S repository](https://github.com/Hado-K3n/bunkerweb/tree/master/docs/K8S/images/Github-repo.png )

Once your managed kubernetes cluster is in place, you need to personnalize your downloaded yaml file to fit your specifications.

### Specify the DNS for your cluster

```
          env:
            - name: KUBERNETES_MODE
              value: "yes"
            # replace with your DNS resolvers
            # e.g. : kube-dns.kube-system.c.cluster.local
            - name: DNS_RESOLVERS
              value: "kube-dns.kube-system.svc.cluster.local"
```

### Change the default values 

You should change the default database password. For example "changeme" to "testor":


e.g:
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

### Change the default UI values

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
              value: "http://bw.lpflov.nodes.c2.gra.k8s.ovh.net/admin/" # specify your URL
            - name: KUBERNETES_MODE
              value: "YES"
            - name: "DATABASE_URI"
              value: "postgresql://bunkerweb:testor@svc-bunkerweb-db:5432/db"
```
It's important to note that your URL needs to end with a "/"

Also important to note, the admin password must contain at least 8 characters, including at least 1 uppercase letter, 1 lowercase letter, 1 number and 1 special character (#@?!$%^&*-).

### Install Hello-world 
We can install an Hello-world deployment for testing purposes
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
spec:
  replicas: 1
  strategy:
    type: Recreate
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
          image: bhargavshah86/kube-test:v0.1
          ports:
          - containerPort: 80
          resources:
            limits:
              memory: 256Mi
              cpu: "250m"
            requests:
              memory: 128Mi
              cpu: "80m"
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```


### Change the default ingress values

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
    bunkerweb.io/bw.lpflov.nodes.c2.gra.k8s.ovh.net_USE_UI: "yes"
    bunkerweb.io/bw.lpflov.nodes.c2.gra.k8s.ovh.net_REVERSE_PROXY_HEADERS_1: "X-Script-Name /admin"
    bunkerweb.io/bw.lpflov.nodes.c2.gra.k8s.ovh.net_INTERCEPTED_ERROR_CODES: "400 404 405 413 429 500 501 502 503 504"
spec:
  rules:
    - host: bw.lpflov.nodes.c2.gra.k8s.ovh.net
      http:
        paths:
          - path: /admin/
            pathType: Prefix
            backend:
              service:
                name: svc-bunkerweb-ui
                port:
                  number: 7000
    - host: hw.lpflov.nodes.c2.gra.k8s.ovh.net
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-world
                port:
                  number: 8080
```

