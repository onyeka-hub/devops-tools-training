# Continuous Deployment with GitLab, Helm, and AWS EKS

This will show you how to deploy a complete CI/CD pipeline on AWS EKS

## Module 1

- Get ready!

- Our sample application

- Deploying our ELS cluster

- Quick Kubernetes review

- Accessing internal services

- DNS, Ingress, Metrics

## Module 2

- Managing stacks with Helm

- ExternalDNS

- Installing Traefik

- Installing metrics-server

- Prometheus and Grafana

- cert-manager

- CI/CD with GitLab

### Get ready!
We're going to set up a whole Continous Deployment pipeline
... for Kubernetes apps
... on a Kubernetes cluster
Ingredients: cert-manager, GitLab, Helm, AWS DNS, EKS, NGINX-INGRESS

### What we're going to do
- Spin up an EKS cluster
- Run a simple test app
- Install a few extras
- Set up GitLab
- Push an app with a CD pipeline to GitLab

### What you need to have
If you want to run this on your own...
- AWS account
- A domain name that you will point to AWS DNS(I got onyeka.ga)
- Local tools to control your Kubernetes cluster:
        kubectl
        helm
- Patience, as many operations will require us to wait a few minutes!

### Why do I need a domain name?
- Because accessing gitlab.onyeka.ga is easier than 102.34.55.67
- Because we'll need TLS certificates (and it's very easy to obtain certs with Let's Encrypt when we have a domain)
- We'll illustrate automatic DNS configuration with ExternalDNS, too! (Kubernetes will automatically create DNS entries in our domain)

### Warning ⚠️💸
- We're going to spin up cloud resources
- Remember to shut them down when you're done!
- In the immortal words of Cloud Economist Corey Quinn: You're charged for what you forget to turn off.

### Deploying our EKS cluster
- If we wanted to deploy Kubernetes manually, what would we need to do? (not that I recommend doing that...)
- Control plane (etcd, API server, scheduler, controllers)
- Nodes (VMs with a container engine + the Kubelet agent; CNI setup)
- High availability (etcd clustering, API load balancer)
- Security (CA and TLS certificates everywhere)
- Cloud integration (to provision LoadBalancer services, storage...)
- And that's just to get a basic cluster! 
- Refer to https://github.com/onyeka-hub/Project-21.git for guidelines

### Managed Kubernetes
- Cloud provider runs the control plane (including etcd, API load balancer, TLS setup, cloud integration)
- We or cloud provider run nodes (the cloud provider generally gives us an easy way to provision them)
- Get started in minutes
- We're going to use AWS EKS Kubernetes Engine

### Creating a cluster
From your command line, using Terraform deploy a version of eks of your choice
Refer to https://github.com/onyeka-hub/terraform-eks-module-official.git

### Quick Kubernetes review
- Let's deploy a simple HTTP server
- And expose it to the outside world!
- Feel free to skip this section if you're familiar with Kubernetes

### Creating a container
On Kubernetes, one doesn't simply run a container

We need to create a "Pod"

A Pod will be a group of containers running together (often, it will be a group of one container)

We can create a standalone Pod, but generally, we'll use a controller (for instance: Deployment, Replica Set, Daemon Set, Job, Stateful Set...)

The controller will take care of scaling and recreating the Pod if needed (note that within a Pod, containers can also be restarted automatically if needed)

### A controller, you said?
We're going to use one of the most common controllers: a Deployment

#### Deployments...
- can be scaled (will create the requested number of Pods)
- will recreate Pods if e.g. they get evicted or their Node is down
- handle rolling updates
- Deployments actually delegate a lot of these tasks to Replica Sets
- We will generally have the following hierarchy:
- Deployment → Replica Set → Pod

#### Exposing the Deployment
- We need to create a Service
- We can use kubectl expose for that 
- For internal use, we can use the default Service type, ClusterIP:
        `kubectl expose deployment web --port=80`
- For external use, we can use a Service of type LoadBalancer:
        `kubectl expose deployment web --port=80 --type=LoadBalancer`

#### Changing the Service type
- We can kubectl delete service web and recreate it
- Or, kubectl edit service web and dive into the YAML
- Or, kubectl patch service web --patch '{"spec": {"type": "LoadBalancer"}}'
- ... These are just a few "classic" methods; there are many ways to do this!

#### Deployment → Pod
- Can we check exactly what's going on when the Pod is created?

- Option 1: watch kubectl get all
        - displays all object types
        - refreshes every 2 seconds
        - puts a high load on the API server when there are many objects

- Option 2: kubectl get pods --watch --output-watch-events
        - can only display one type of object
        - will show all modifications happening (à la tail -f)
        - doesn't put a high load on the API server (except for initial display)

### Accessing internal services
- How can we temporarily access a service without exposing it to everyone?
- kubectl proxy: gives us access to the API, which includes a proxy for HTTP resources
- kubectl port-forward: allows forwarding of TCP ports to arbitrary pods, services, ...

#### kubectl proxy in theory
- Running kubectl proxy gives us access to the entire Kubernetes API
- The API includes routes to proxy HTTP traffic
- These routes look like the following:
        `/api/v1/namespaces/<namespace>/services/<service>/proxy`
- We just add the URI to the end of the request, for instance:
        `/api/v1/namespaces/<namespace>/services/<service>/proxy/index.html`
- We can access services and pods this way

#### kubectl proxy in practice
- Let's access the web service through kubectl proxy
- Run an API proxy in the background:
        `kubectl proxy &`
- Access the web service:
        `curl localhost:8001/api/v1/namespaces/default/services/web/proxy/`
- Terminate the proxy:
        `kill %1`

#### kubectl port-forward in theory
- What if we want to access a TCP service?
- We can use kubectl port-forward instead
- It will create a TCP relay to forward connections to a specific port (of a pod, service, deployment...)
- The syntax is:
        `kubectl port-forward service/name_of_service local_port:remote_port`

- If only one port number is specified, it is used for both local and remote ports  

#### kubectl port-forward in practice
- Let's access our remote NGINX server
- Forward connections from local port 1234 to remote port 80:
        `kubectl port-forward svc/web 1234:80 &`
- Connect to the NGINX server:
        `curl localhost:1234`
- Terminate the port forwarder:
        `kill %1`

### DNS, Ingress, Metrics
- We got a basic app up and running
- We accessed it over a raw IP address
- Can we do better? (i.e. access it with a domain name!)
- How much resources is it using?

#### DNS
- We'd like to associate a fancy name to that LoadBalancer Service (e.g. nginx.onyeka.ga → A.B.C.D)
- option 1: manually add a DNS record
- option 2: find a way to create DNS records automatically
- We will install ExternalDNS to automate DNS records creatoin
- ExternalDNS supports AWS DNS and dozens of other providers

#### Ingress
- What if we have multiple web services to expose?
- We could create one LoadBalancer Service for each of them
- This would create a lot of cloud load balancers (and they typically incur a cost, even if it's a small one)
- Instead, we can use an Ingress Controller
- Ingress Controller = HTTP load balancer / reverse proxy
- Put all our HTTP services behind a single LoadBalancer Service
- Can also do fancy "content-based" routing (using headers, request path...)
- We will install nginx as our Ingress Controller

#### Metrics
- How much resources are we using right now?
- When will we need to scale up our cluster?
- We need metrics!
- We're going to install the metrics server
- It's a very basic metrics system (no retention, no graphs, no alerting...)
- But it's lightweight, and it is used internally by Kubernetes for autoscaling

### What's next
- We're going to install all these components
- Very often, things can be installed with a simple YAML file
- Very often, that YAML file needs to be customized a little bit (add command-line parameters, provide API tokens...)
- Instead, we're going to use Helm charts
- Helm charts give us a way to customize what we deploy
- Helm can also keep track of what we install (for easier uninstall and updates)

### Helm concepts
- helm is a CLI tool
- It is used to find, install, upgrade charts
- A chart is an archive containing templatized YAML bundles
- Charts are versioned
- Charts can be stored on private or public repositories

#### Charts and repositories
A repository (or repo in short) is a collection of charts. It's just a bunch of files, (they can be hosted by a static HTTP server, or on a local directory).

We can add "repos" to Helm, giving them a nickname. The nickname is used when referring to charts on that repo. (for instance, if we try to install hello/world, that means the chart world on the repo hello; and that repo hello might be something like https://blahblah.hello.io/charts/)

#### How to find charts
Go to the Artifact Hub (https://artifacthub.io). Or use helm search hub ... from the CLI. Let's try to find a Helm chart for something called "OWASP Juice Shop"!. (it is a famous demo app used in security challenges)

#### Finding charts from the CLI
We can use helm search hub <keyword>

Look for the OWASP Juice Shop app:

`helm search hub owasp juice`

Since the URLs are truncated, try with the YAML output:

`helm search hub owasp juice -o yaml`

Then go to → https://artifacthub.io/packages/helm/seccurecodebox/juice-shop

#### Finding charts on the web
We can also use the Artifact Hub search feature. Go to https://artifacthub.io/. In the search box on top, enter "owasp juice". Click on the "juice-shop" result (not "multi-juicer" or "juicy-ctf")

#### Installing the chart
- Click on the "Install" button, it will show instructions
- First, add the repository for that chart:
        `helm repo add juice https://charts.securecodebox.io`

- Then, install the chart:
        `helm install my-juice-shop juice/juice-shop`

Note: it is also possible to install directly a chart, with --repo https://...

#### Charts and releases
"Installing a chart" means creating a release. In the previous exemple, the release was named "my-juice-shop". We can also use --generate-name to ask Helm to generate a name for us.

List the releases:
        `helm list`

Check that we have a my-juice-shop-... Pod up and running:
        `kubectl get pods`

#### Viewing resources of a release
This specific chart labels all its resources with a release label. We can use a selector to see these resources

List all the resources created by this release:
        `kubectl get all --selector=app.kubernetes.io/instance=my-juice-shop`
Note: this label wasn't added automatically by Helm.
It is defined in that chart. In other words, not all charts will provide this label.

#### Configuring a release
By default, juice/juice-shop creates a service of type ClusterIP. We would like to change that to a NodePort. We could use kubectl edit service my-juice-shop, but  our changes would get overwritten next time we update that chart!

Instead, we are going to set a value. Values are parameters that the chart can use to change its behavior. Values have default values. Each chart is free to define its own values and their defaults.

#### Checking possible values
We can inspect a chart with helm show or helm inspect
Look at the README for the app:
        `helm show readme juice/juice-shop`

Look at the values and their defaults:
        `helm show values juice/juice-shop`

The values may or may not have useful comments. The readme may or may not have (accurate) explanations for the values. (If we're unlucky, there won't be any indication about how to use the values!)

#### Setting values
Values can be set when installing a chart, or when upgrading it. We are going to update my-juice-shop to change the type of the service.

Update my-juice-shop:
        `helm upgrade my-juice-shop juice/juice-shop --set service.type=NodePort`
Note that we have to specify the chart that we use (juice/my-juice-shop), even if we just want to update some values.

We can set multiple values. If we want to set many values, we can use -f/--values and pass a YAML file with all the values.

All unspecified values will take the default values defined in the chart.

#### Connecting to the Juice Shop
Let's check the app that we just installed.

Check the node port allocated to the service:
        `kubectl get service my-juice-shop`

PORT=$(kubectl get service my-juice-shop -o jsonpath={..nodePort})

Connect to it:
        `curl localhost:$PORT/`

![juice-shop page](./images/1-page.PNG)

### ExternalDNS
ExternalDNS will automatically create DNS records from Kubernetes resources
        - Services (with the annotation external-dns.alpha.kubernetes.io/hostname)
        - Ingresses (automatically)

It requires a domain name (obviously)... And that domain name should be configurable through an API. As of April 2021, it supports a few dozens of providers. We're going to use AWS DNS.

#### Deploying ExternalDNS
The ExternalDNS documentation has a tutorial for AWS. It's basically a lot of YAML! That's where using a Helm chart will be very helpful. There are a few ExternalDNS charts available out there. We will use the one from  https://kubernetes-sigs.github.io/external-dns/ .

#### How we'll install things with Helm
We will install each chart in its own namespace. (this is not mandatory, but it helps to see what belongs to what).

We will use helm upgrade --install instead of helm install, (that way, if we want to change something, we can just re-run the command)

We will use the --create-namespace and --namespace ... options

To keep things boring and predictible, if we are installing chart xyz:
        - we will install it in namespace xyz
        - we will name the release xyz as well

#### Installing ExternalDNS
First, let's add the Externaldns repo:

helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/

To set up externaldns on aws eks cluster, please refer to this documentation for the neccessary steps like permission, externaldns service account, etc before installation.

##### Short description
ExternalDNS is a pod that runs in your Amazon EKS cluster. To use ExternalDNS as a plugin with Amazon EKS, set up AWS Identity and Access Management (IAM) permissions. These permissions must allow Amazon EKS access to Amazon Route 53.

**Note**: Before starting the following resolution, make sure that a domain name and a Route 53 hosted zone exist.
#### Steps
1. Set up IAM permissions to give the ExternalDNS pod permissions to create, update, and delete Route 53 records in your AWS account.

From the aws console crerate an IAM policy with the policy below and name it 'externaldns':
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```
**Note**: You can also adjust the preceding policy to allow updates to explicit hosted zone IDs.

2. Use the preceding policy to create an IAM role for the service account:
```
eksctl create iamserviceaccount --name SERVICE_ACCOUNT_NAME --namespace NAMESPACE --cluster CLUSTER_NAME --attach-policy-arn IAM_POLICY_ARN --approve
```
Or as in my case
```
eksctl create iamserviceaccount --name externaldns --namespace externaldns --cluster CLUSTER_NAME --attach-policy-arn arn:aws:iam::958217526797:policy/external-dns --approve
```

To check the name of your service account, run the following command:
```
kubectl get sa
```
Example output:
```
NAME           SECRETS   AGE
default        1         23h
external-dns   1         23h
```

In the preceding example output, external-dns is the name that was given to the service account when it was created. Make sure you have eksctl intalled on your system.

3. Then, install ExternalDNS:
```
helm upgrade --install external-dns external-dns/external-dns \
  --namespace external-dns --create-namespace \
  --set serviceAccount.create=false,serviceAccount.name=extternaldns
```
Or
```
helm upgrade --install external-dns external-dns/external-dns --set serviceAccount.create=false,serviceAccount.name=extternaldns --namespace external-dns
```
4.    Verify that the deployment was successful:
```
kubectl get deployments
```
Example output:
```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
external-dns   1/1     1            1           85m
```
You can also check the logs to verify that the records are up to date:
```
kubectl logs external-dns-9f85d8d5b-sx5fg
```
Example output:
```
....
....
time="2022-02-10T20:22:02Z" level=info msg="Instantiating new Kubernetes client"
time="2022-02-10T20:22:02Z" level=info msg="Using inCluster-config based on serviceaccount-token"
time="2022-02-10T20:22:02Z" level=info msg="Created Kubernetes client https://10.100.0.1:443"
time="2022-02-10T20:22:09Z" level=info msg="Applying provider record filter for domains: [<yourdomainname>.com. .<yourdomainname>.com.]"
time="2022-02-10T20:22:09Z" level=info msg="All records are already up to date"
....
....
```

5. Verify that ExternalDNS is working
- Create a service that's exposed as LoadBalancer and that can be routed externally through the domain name that's hosted on Route 53 or edit the juice-shop service to a type LoadBalancer and annotate it.

Manifest:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: DOMAIN_NAME
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: LoadBalancer
  selector:
    app: nginx
```
```
kubectl apply SERVICE_MANIFEST_FILE_NAME.yaml
```

- Check that the NGINX service was created with the LoadBalancer type:
```
kubectl get svc
```
Example output:
```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)        AGE
kubernetes   ClusterIP      10.100.0.1      <none>                                                                   443/TCP        26h
nginx        LoadBalancer   10.100.234.77   a1ef09255d52049f487e05b4f74faea6-954147917.us-west-1.elb.amazonaws.com   80:30792/TCP   74m
```

Note: The service automatically creates a Route 53 record for the hosted zone.

Check the logs to verify that the Route 53 record was created:
```
kubectl logs external-dns-9f85d8d5b-sx5fg
```
Example output:
```
...
...
...
time="2022-02-10T21:22:43Z" level=info msg="Applying provider record filter for domains: [<domainname>.com. .<domainname>.com.]"
time="2022-02-10T21:22:43Z" level=info msg="Desired change: CREATE <domainname>.com A [Id: /hostedzone/Z01155763Q6AN7CEI3AP6]"
time="2022-02-10T21:22:43Z" level=info msg="Desired change: CREATE <domainname>.com TXT [Id: /hostedzone/Z01155763Q6AN7CEI3AP6]"
time="2022-02-10T21:22:43Z" level=info msg="2 record(s) in zone xxx.com. [Id: /hostedzone/Z01155763Q6AN7CEI3AP6] were successfully updated"
time="2022-02-10T21:23:43Z" level=info msg="Applying provider record filter for domains: [<domainname>.com. .<domainname>.com.]"
time="2022-02-10T21:23:43Z" level=info msg="All records are already up to date"
...
...
...
```

![externaldns successful](./images/external-dns-successful.PNG)

For more information and examples for ExternalDNS, see Setting up ExternalDNS for services on AWS (on the GitHub website) and Set up ExternalDNS (on the Kubernetes website).

### Installing Nginx-Ingress
Traefik is going to be our Ingress Controller

Let's install it with a Helm chart, in its own namespace

First, let's add the Traefik chart repository:

helm repo add traefik https://helm.traefik.io/traefik
Then, install the chart:

helm upgrade --install traefik traefik/traefik \
    --create-namespace --namespace traefik \
    --set "ports.websecure.tls.enabled=true"
(that option that we added enables HTTPS, it will be useful later!)