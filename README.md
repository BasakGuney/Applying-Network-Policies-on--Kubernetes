# Applying Network Policies on Kubernetes

> Network Policies in Kubernetes provide a way to manage the flow of network traffic within a cluster. They enable you to specify which Pods can communicate with each other. Implementing these policies helps isolate applications, preventing unauthorized communication between them, and minimizing potential damage if an application is compromised.

## 1. Restrict All Traffic Allow DNS Only
```txt
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-dns
  namespace: 
spec: <namespace>
  podSelector: {}
  policyTypes:
  - Egress
  - Ingress
  egress:
  - ports:
    - port: 53
      protocol: TCP
    - port: 53
      protocol: UDP
```
Create allow-only-dns.yaml and copy the content above then save and exit.  
<br>


```bash
kubectl apply -f allow-only-dns.yaml
```
Run the command above to apply the NetworkPolicy.

Once you have applied this NetworkPolicy, all other traffic other than DNS are restricted. To allow access between pods or services you can use the followings:

## 2. Allowing Pairwise Access

> Consider two nginx pods which are created by the following YAMLs.

nginx1.yaml:
```txt
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
  labels:
    app: nginx1
spec:
  containers:
  - name: nginx1
    image: nginx:latest
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx1
spec:
  selector:
    app: nginx1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```
<br>

nginx2.yaml:
```txt
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
  labels:
    app: nginx2
spec:
  containers:
  - name: nginx2
    image: nginx:latest
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx2
spec:
  selector:
    app: nginx2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```
Since you applied the allow-dns-only policy above, these pods can't be able to access
each other. You can check it by doing the followings:
```bash
kubectl exec -n <namespace> nginx1 -- curl nginx2 --verbose
````
and 

```bash
kubectl exec -n <namespace> nginx2 -- curl nginx1 --verbose
```
You can see that the IP of the pod services can be resolved when you use curl with --verbose flag. It shows that DNS resolution is enabled.

Now, consider you want to allow access from nginx1 to nginx2. Then you have to allow outgoing traffic to nginx2 in nginx1 and allow incoming traffic from nginx1 in nginx2.  
<br>

To allow outgoing traffic to nginx2 in nginx1:

```txt
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx1-outgoing
  namespace: <deneme>
spec:
  podSelector:
    matchLabels:
      app: nginx1
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: nginx2
```
<br>

To allow outgoing traffic from nginx1 in nginx2: 

```txt 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-auto-nginx2
  namespace: app-auto
spec:
  podSelector:
    matchLabels:
      app: nginx2
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: nginx1
```
<br>

> These policies use the pod labels to match the pod that will be affected. You have to be sure that your pods have "app" label. The labels can differ according to how you create the pods. You can check your pod labels by:

```bash
 kubectl get pod -n <namespace> <pod-name> --show-labels
```

If you don't have "app" label you can use other labels of your pod in the matchLabels part in the Network Policy YAML.


After you apply the policies above, you have to be able to access nginx2 from nginx2. You can test it by:
```bash
kubectl exec -n <namespace> nginx1 -- curl nginx2 --verbose
```
Now you have to be able to see "Connected to nginx2" line in the output.  

<br>

Also you can allow traffic of IP blocks using the following format:
```txt
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: policy-for-IP-blocks
  namespace: <namespace>  # Change to your target namespace
spec:
  podSelector:
    matchLabels:
      app: <app>
  policyTypes:
  - Egress
  - Ingress
  egress:
    - to:
        - ipBlock:
            cidr: X.X.X.X/32 # Allow access from the specific IP address
      ports:
        - protocol: TCP
          port: <port> # Allow access on port 
  ingress:
    - from:
        - ipBlock:
            cidr: X.X.X.X/32 # Allow access from the specific IP address
      ports:
        - protocol: TCP
          port: <port> # Allow access on port 
```
