# Voting App on EKS

This is my Project 2 for the Ironhack DevOps bootcamp. In the capstone, I deployed this same polyglot voting app, Python vote UI, Node.js result UI, .NET worker, Redis, and Postgres, across a three-tier EC2 architecture using Terraform for provisioning and Ansible for configuration. This project takes the same app and moves it onto a managed Kubernetes cluster on EKS instead, with GitHub Actions handling the build and deploy pipeline automatically.

## Live demo

    Vote: https://voting.flinktrade.com/vote/
    Result: https://voting.flinktrade.com/result/

This points at a live EKS cluster, so it only works while the cluster is running. If the links are down, it likely means the cluster was torn down after grading or demo to avoid ongoing AWS costs.

## Why EKS

We'd already run Kubernetes locally on Minikube earlier in the bootcamp, but Minikube is a single node on your own laptop. EKS gave me a chance to work with an actual multi-node cluster, IAM permissions, and a managed control plane, closer to what this would look like in a real job.

## Stack

I provisioned the cluster with eksctl, which handles the VPC, nodegroup, and IAM roles for you under the hood through CloudFormation. Once the cluster exists, everything else is kubectl. I used Helm to install the NGINX Ingress Controller, and GitHub Actions handles the CI/CD side: building the images, pushing them to Docker Hub, and applying the manifests.

## Setting up the project

I set up the folder structure first, one subfolder per service under k8s, so the manifests stay organized instead of piling up loose in the root.

    mkdir -p k8s/redis k8s/postgres k8s/vote k8s/result k8s/worker k8s/ingress
    mkdir -p .github/workflows docs/screenshots

![Repo folder structure](docs/screenshots/01-repo-structure.png)

## Provisioning the EKS cluster

Before any manifests can be applied, I need an actual cluster. I used eksctl since it handles the VPC, subnets, IAM roles, and worker nodes in one pass.

I kept the nodegroup small, two t3.medium nodes, since this runs on a shared classroom AWS account. Two nodes is still enough to show the services spreading across multiple machines, not just Minikube with extra steps.

    eksctl create cluster -f cluster-config.yaml --profile nazmul

This took about 15 minutes. eksctl built a CloudFormation stack behind the scenes: the EKS control plane, a VPC across two availability zones, IAM roles, and the EC2 nodes. It also wrote my kubeconfig automatically, so kubectl was ready immediately.

![EKS cluster created, nodes ready](docs/screenshots/02-cluster-created.png)

## Confirming the cluster is live

Once the cluster finished creating, I wanted proof it actually worked rather than trusting the eksctl output alone. Checking the nodes directly shows both EC2 instances registered with Kubernetes and marked Ready, running the Kubernetes version I asked for.

    kubectl get nodes -o wide

This confirms two things at once: kubectl is correctly pointed at the new cluster, and both nodes joined the cluster successfully and are healthy enough to run workloads. Nothing in Phase 2 can start until this comes back clean.

![Both nodes ready](docs/screenshots/03-nodes-ready.png)

## Deploying Redis and Postgres

Redis and Postgres each got a Deployment to keep a container running, and a Service to give it a stable internal name. Redis came up immediately. Postgres also needed a Secret for its database credentials and a PersistentVolumeClaim for durable storage.

The PVC got stuck in Pending, tracing it through kubectl describe showed the pod could not schedule because the claim was unbound. The EBS CSI driver, which actually provisions AWS storage for Kubernetes, was not installed by default when the cluster was created. After installing it as an eksctl addon, the PVC was still not binding, since the only existing StorageClass on this shared classroom cluster used the older in-tree AWS provisioner rather than the CSI driver, and nothing pointed at it as default.

Fixed it by creating a StorageClass explicitly using the CSI provisioner and referencing it by name in the PVC directly, rather than relying on a default. Once applied, the PVC bound immediately and Postgres started.

    kubectl apply -f k8s/redis/
    kubectl apply -f k8s/postgres/

![Redis and Postgres running](docs/screenshots/04-redis-postgres-running.png)

## Deploying vote, result, and worker

All three services pulled from images I had already built and pushed to Docker Hub during the capstone, so this phase was mostly about connecting them correctly to Redis and Postgres through environment variables. result and worker both needed their database host and credentials overridden to match the actual Postgres service and Secret in this cluster, rather than the generic defaults baked into the Dockerfiles.

vote crash looped on startup with a Python error trying to convert REDIS_PORT into a number. The cause was not the app or the manifest, it was Kubernetes itself. Kubernetes automatically injects an environment variable for every Service in the namespace, named after the service, and since the Redis Service is named redis, it injected its own REDIS_PORT as a full connection URI rather than a plain port number, silently overriding the plain integer the Dockerfile expected. Fixed it by setting REDIS_PORT explicitly in the pod spec, which takes precedence over the auto-injected one.

    kubectl apply -f k8s/vote/
    kubectl apply -f k8s/result/
    kubectl apply -f k8s/worker/

![All five pods running](docs/screenshots/05-all-pods-running.png)

## Installing the NGINX Ingress Controller

Every service so far only exists inside the cluster, nothing outside AWS could reach vote or result yet. The Ingress Controller is what exposes them to the internet and routes different URL paths to the right backend service. I used Helm to install it rather than writing the underlying resources by hand, since it is a fairly large bundle, RBAC rules, a controller Deployment, and a LoadBalancer Service.

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

On AWS, a LoadBalancer type Service is not just an internal thing, EKS automatically provisions a real Elastic Load Balancer for it. Once the controller pod was running, kubectl get svc showed the ELB's external hostname, which is the actual public entry point into the cluster.

![Ingress controller live with ELB hostname assigned](docs/screenshots/06-ingress-controller-live.png)

## Routing traffic with the Ingress resource

With the controller running, the last piece was telling it how to route incoming paths to the right service. The Ingress resource maps requests starting with /vote to vote-service and requests starting with /result to result-service, both on port 80. Testing with curl against the load balancer hostname confirmed both routes work end to end, from the public internet through the ELB, into the Ingress controller, and to the correct pod.

I initially used the older kubernetes.io/ingress.class annotation, which worked but is deprecated, and switched to the current spec.ingressClassName field instead, which is why the CLASS column below shows nginx rather than none.

    kubectl apply -f k8s/ingress/

![Ingress routing confirmed working](docs/screenshots/07-ingress-routing-working.png)

## GitHub Actions CI/CD pipeline

The last piece was automating the manual flow, build, push, apply, into a pipeline that runs on every push to main. The workflow logs into Docker Hub, builds and pushes all three images, configures AWS credentials, points kubectl at the EKS cluster the same way eksctl did locally, applies the manifests, then restarts each deployment so it actually pulls the freshly pushed latest image rather than keeping whatever was already running.

First run failed since the source folders for vote, result, and worker had never actually been pushed to the repo, only their Docker images were, from manual builds earlier. Second failure was a Docker Hub access token with the wrong scope, it could log in but not push. Regenerating the token with read, write, and delete access fixed it.

![CI/CD pipeline running green](docs/screenshots/08-cicd-pipeline-success.png)

Checking pod ages afterward confirmed the whole pipeline reached the cluster, not just Docker Hub, since a fresh rollout restart recreates every pod with a brand new age.

![Pods recreated by the pipeline](docs/screenshots/09-pipeline-verified-in-cluster.png)

## Fixing sub-path routing in the vote app

Once the Ingress rewrite was in place, the vote page loaded but was unstyled, and clicking a vote button led to an error. The vote app was originally written assuming it lives at the root of its own server, exactly how it ran in the capstone, so its HTML used a base tag and absolute paths that always pointed back to the domain root regardless of the actual URL. Behind an Ingress at a /vote sub-path, that assumption breaks.

Fixed the template first, removing the base tag and switching the form action and stylesheet link to relative paths. That surfaced a second issue: Flask only had a route for /vote, not /vote/, and a trailing slash is what makes relative paths resolve correctly under a sub-path, so a plain GET to /vote/ was 404ing. Added a matching route. That in turn surfaced a third issue, Flask serves static files at /static/ by default, not /vote/static/, so the browser was asking for a path Flask did not recognize. Added an explicit route to serve static assets under /vote/static/ as well.

Three separate layers, the Ingress rewrite, Flask routing, and static file serving, all had to agree on the same path structure before voting and styling both worked.

![Voting app working end to end through the Ingress](docs/screenshots/10-final-demo.png)

## HTTPS with cert-manager and Cloudflare

The last addon was serving the app over a real domain with a trusted certificate instead of a bare HTTP load balancer hostname. Installed cert-manager via Helm, created a ClusterIssuer configured for Lets Encrypt using DNS-01 validation through Cloudflare, since flinktrade.com DNS is managed there rather than Route 53. Added a CNAME for voting.flinktrade.com pointing at the ELB, and updated the Ingress with a tls block and the cert-manager cluster-issuer annotation so a Certificate gets requested automatically.

Hit a few real issues along the way. The ClusterIssuer initially failed since the email field was left as an unreplaced placeholder. The DNS-01 challenge then failed with an invalid token error, caused by a trailing newline that got baked into the Cloudflare API token when it was saved through a text file, fixed by stripping it before creating the secret. Once fixed, the certificate issued successfully.

With HTTPS working, voting through the new domain surfaced one more bug: the result page uses Socket.IO for live updates, and its server was hardcoded to expect connections on /result/socket.io, a path that no longer existed once the Ingress rewrite stripped the /result prefix before requests reached the pod. The client needed that prefixed path to route correctly through the Ingress, but the server needed to expect the stripped version. Removing the hardcoded path from the server side, so it just used the default, resolved the mismatch and results now update live over the real domain.

    https://voting.flinktrade.com/vote/
    https://voting.flinktrade.com/result/

![Voting app live over HTTPS on a real domain, results updating correctly](docs/screenshots/11-https-cert-live.png)

## Summary

This project took the polyglot voting app from the capstone and moved it from a three-tier EC2 setup onto Amazon EKS, with a full GitHub Actions pipeline handling build and deploy on every push. Beyond the core assignment, it also includes two optional addons: Kubernetes Secrets for database credentials, and HTTPS through cert-manager and Cloudflare, serving the app on a real domain with a trusted Let's Encrypt certificate rather than a bare load balancer hostname.

If I were doing this again, I would write the vote and result apps to be path-agnostic from the start, using relative paths and a configurable base path rather than assuming they would always be served from the root. Almost every routing bug in this project traced back to that one assumption.

## Problems faced and how they were fixed

Several real issues came up during this build, each one taught something about how EKS actually behaves versus how it looks on paper.

The Postgres PersistentVolumeClaim stayed stuck in Pending because the EBS CSI driver was not installed by default when the cluster was created, and the one StorageClass that did exist used an older provisioner. Installed the CSI driver as an addon and created a StorageClass explicitly pointing at it.

The vote app crash looped on startup because Kubernetes automatically injects a REDIS_PORT environment variable for any Service named redis, and that auto-injected value collided with the plain port number the app expected. Fixed by setting REDIS_PORT explicitly in the pod spec, which takes precedence.

Once the Ingress was routing by path prefix, the vote page lost its styling and its vote button stopped working. The app's HTML used a base tag and absolute paths that always resolved to the domain root, and Flask had no route for the trailing-slash version of /vote, nor a way to serve static files under that sub-path. Fixed all three: removed the base tag, added a trailing-slash route, and added an explicit static file route under /vote/static/.

The GitHub Actions pipeline failed twice on its first real runs. Once because the vote, result, and worker source folders had only ever been built and pushed manually, never actually committed to the repo. Once because the Docker Hub access token was scoped without push permission. Fixed by committing the missing folders and regenerating the token with the correct scope.

Setting up HTTPS surfaced a few more issues. The ClusterIssuer failed to register with Let's Encrypt because the contact email was left as an unreplaced placeholder. The DNS-01 challenge then failed because the Cloudflare API token had a trailing newline baked in from being saved through a text file, which Cloudflare's API rejected outright. Fixed by stripping the newline before creating the secret.

With HTTPS working, the result page stopped showing live updates. Its Socket.IO client was correctly told to connect on /result/socket.io so the Ingress would route it, but the server was hardcoded to expect that same literal path, which no longer existed once the Ingress rewrite stripped the /result prefix before the request reached the pod. Fixed by removing the hardcoded path from the server side so it used the default, matching what actually arrives after the rewrite.

Last one: after voting, the confirmation checkmark stopped appearing. The page's jQuery script tag was loaded over plain http, and browsers silently block that kind of mixed content on an https page, so the script never ran. Fixed by switching it to https.
