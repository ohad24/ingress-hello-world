# Kubernetes ingress hello world

Little example for to deploy 2 deployments, 2 services and ingress on minikube and GKE.  
This example based on [ingress-minikube](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/) docs.  
[GKE](https://cloud.google.com/kubernetes-engine) steps a optional for extended deployments.


## Prerequisite
You should have [minikube](https://minikube.sigs.k8s.io/docs/start/) (in minikube deployment) and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) installed.  
The `k` keyword is alias for `kubectl` command.

## Minikube
Start the cluster with named **mini1**, 3 nodes with ingress addon.
```bash
minikube start -p mini1 -n 3 --addons=ingress
```
Configure the context in your `kubectl` if needed. Then, check the nodes.
```bash
❯ k get nodes
NAME        STATUS   ROLES                  AGE     VERSION
mini1       Ready    control-plane,master   2m25s   v1.20.7
mini1-m02   Ready    <none>                 47s     v1.20.7
mini1-m03   Ready    <none>                 28s     v1.20.7
```

Create namespace
```bash
k apply -f ns.yaml
```

Create deployments
```bash
k apply -f deployments/
```

Create services (`type: ClusterIP`)
```bash
k apply -f services/
```

Create ingress
```
k apply -f minikube_ingress.yaml
```
The ingress will deploy on `ingress-nginx` namespace.  
You can check it's deployment by run the following command.  
**It will be ready as soon as you see the ADDRESS field populate with valid IP address, after 1-2 minutes.**
```bash
❯ k get ingress -o wide -A
NAMESPACE       NAME               CLASS    HOSTS              ADDRESS        PORTS   AGE
hello-ingress   minikube-ingress   <none>   hello-world.info   192.168.49.2   80      30s
```


In case you running minikube on WSL, you will have to forward the ingress to the host machine using the `minikube service` command.  
In this example will forward port 80 to 38633 and port 443 to 40685.  
**Each time you will start new minikube service it will pick random ports.**
```bash
❯ minikube service ingress-nginx-controller -n ingress-nginx -p mini1
|---------------|--------------------------|-------------|---------------------------|
|   NAMESPACE   |           NAME           | TARGET PORT |            URL            |
|---------------|--------------------------|-------------|---------------------------|
| ingress-nginx | ingress-nginx-controller | http/80     | http://192.168.49.2:31566 |
|               |                          | https/443   | http://192.168.49.2:32583 |
|---------------|--------------------------|-------------|---------------------------|
🏃  Starting tunnel for service ingress-nginx-controller.
|---------------|--------------------------|-------------|------------------------|
|   NAMESPACE   |           NAME           | TARGET PORT |          URL           |
|---------------|--------------------------|-------------|------------------------|
| ingress-nginx | ingress-nginx-controller |             | http://127.0.0.1:38633 |
|               |                          |             | http://127.0.0.1:40685 |
|---------------|--------------------------|-------------|------------------------|
🎉  Opening service ingress-nginx/ingress-nginx-controller in default browser...
👉  http://127.0.0.1:38633
🎉  Opening service ingress-nginx/ingress-nginx-controller in default browser...
👉  http://127.0.0.1:40685
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it.
```
This ingress with deploy and listen to `hello-world.info` domain, so you will have to add the following record to your `hosts` file.
```
127.0.0.1			hello-world.info
```
Test this deployment VIA `curl` command.  
app1:
```bash
❯ curl hello-world.info:38633
Hello, world!
Version: 1.0.0
Hostname: hello-app-66c68bf5f7-7pp5g
```
app2:
```bash
❯ curl hello-world.info:38633/v2
Hello, world!
Version: 2.0.0
Hostname: hello-app2-676db4445d-4wjkm
```
app2 with https:
```bash
❯ curl -k https://hello-world.info:40685/v2
Hello, world!
Version: 2.0.0
Hostname: hello-app2-676db4445d-k8264
```

## GKE
**This following steps will use cloud resources and will charge you for that. Make sure you clean up the resources after finish to test.**  
First you will need to create external IP address.
```bash
gcloud compute addresses create hello-addr --global --ip-version IPV4
```

You can check the resource with those commands
```bash
gcloud compute addresses list
gcloud compute addresses describe hello-addr --global
```

Create the SSL files (private key and chain file). Then create the TLS secrets.
```bash
# generate SSL files
❯ openssl req -newkey rsa:2048 \
    -x509 \
    -sha256 \
    -days 3650 \
    -nodes \
    -out fullchain.pem \
    -keyout privkey.pem\
    -subj "/C=US/ST=test/L=test1/O= /OU= /CN=example.com"
Generating a RSA private key
.....+++++
...........................................................+++++
writing new private key to 'privkey.pem'

# create gcp secret
❯ create secret tls web-tls \
    --cert fullchain.pem --key privkey.pem -n hello-ingress
secret/web-tls created

# generate the yaml from the culuster
# k get secrets -n hello-ingress web-tls -o yaml > web-tls-secret.yaml
```

Then, deploy namespace, deployments and services.
```bash
❯ k apply -f ns.yaml
namespace/hello-ingress created
❯ k apply -f depoyments
deployment.apps/hello-app created
deployment.apps/hello-app2 created
❯ k apply -f services
service/svc created
service/svc2 created
```

Finally, deploy the GKE ingress.  
The command will configure the ingress for HTTPS connections, with auto redirect the HTTP to HTTPS.
```bash
❯ k apply -f gke_ingress.yaml
ingress.networking.k8s.io/gke-hello-ingress created
frontendconfig.networking.gke.io/lb-http2https created
```