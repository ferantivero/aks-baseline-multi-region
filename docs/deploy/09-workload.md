# Deploy the workload

The cluster now has [Traefik configured with a TLS certificate](./08-secret-managment-and-ingress-controller.md). The last step in the process is to deploy the workload, which is the ASP.NET Core Docker web app. The app demonstrates the system's functions.

## Steps

> :book: The Contoso app team is about to conclude this journey, but they need an app to test their new infrastructure. For this task they've selected the [ASP.NET Core Docker sample web app](https://github.com/dotnet/dotnet-docker/tree/main/samples/aspnetapp). The business unit (BU0001) has approved the creation of a second cluster that could help balance the traffic, but mainly to serve as a hot backup. They are a bit worried about the required engineering effort though. The same application (Application ID: a0042) is about to span into multiple clusters, across multiple regions, so there is a desire to find a good mechanism for common configuration management.
>
> With that in mind, the app team is looking at which *federation* approaches they could follow to run different instances of the same app in different clusters. They know that at this point things must be kept simple. In fact, they could run these two application instances (Applications IDs: `a0042-03` and `a0042-04`) from the two regional clusters with just a bunch of scripts. But they want to be sure that the selected approach is not going to add impediments that could prevent them from scaling out their fleet of clusters if there was a future requirement to support more regions.
>
> Depending on how federation is implemented, it could open a door in which a single command execution has an instant ripple effect into all your clusters. While running clusters separately like silos could avoid such ripple effects, the tradeoff in complexity is high and will affect the ability to scale the number of clusters in the future. They know that there is [specialized tooling](https://github.com/kubernetes-sigs/kubefed) to manage a centralized control plane, which pushes the workload's behavior in response to special events like a regional outage. However, the Contoso team wants to proceed with caution in this area for now.
>
> Given those constraints, this reference implementation provides a middle ground solution in which an organization could build the basis for the future without a heavyweight solution for just two clusters. Therefore, the recommendation is to manage the workload manifests separately for each instance from a central *federation* Git repository in combination with a CI/CD pipeline. In this reference implementation, we don't create a separarate federation repository.

![Diagram depicting the federation approach for the proposed cluster fleet topology, which runs different instances of the same application in each cluster.](./images/aks-federation.png)

> :bulb: Federation repos could be a monorepo or multiple repos. In this reference implementation, the workload manifests are shipped together from a single repo.

1. Deploy the ASP.NET Core Docker sample web app in the AKS cluster deployed into the first region (East US 2).

   The workload definition includes of a pod disruption budget rule, ingress configuration, and pod affinity and anti-affinity rules for your reference.

   ```bash
   kubectl apply -f ./workload/aspnetapp-deploy.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   kubectl apply -f ./workload/aspnetapp-ingress.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   ```

1. Deploy another instance of the same workload in the second AKS cluster deployed to the second region (Central US). Additionally, enable some canary testing from the second cluster.

   > :book: The app team is now more confident than ever with its second regional AKS cluster. The team follows an active/active availability strategy, and it is known that the major stream of clients comes from East US 2. Therefore, they realize they have some idle resources most of the time from Central US. This makes them to consider starting some canary testing in that specific region, because this approach introduces a very low risk for normal operations. By using a weighted load balanced strategy, most of the time client requests will be served by the existing ASP.NET workload app, but occasionally an experimental new workload version will be used (running on top of the new: Ubuntu Chiseled Images). From this experimentation, the app team wants to evaluate the deployment size of their workload using this new type of "distroless" container image as well as reduce the attack surface by including only the minimal set of packages required to run .NET applications. This will allow them to plan ahead before a full migration.

   > :warning: Note that this canary testing approach is not recommended for organizations that are operating critical production systems at peak utilization. But this approach is a good example of one of the options available when you extend your architecture to multiple clusters, and how you can make use of idle resources with care.

   ```bash
   kubectl apply -f ./workload/aspnetapp-deploy.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   kubectl apply -f ./workload/aspnetapp-ingress.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   kubectl apply -f ./workload/aspnetapp-canary-deploy.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   kubectl apply -f ./workload/aspnetapp-canary-ingress.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   ```

1. Wait until both regions are ready to process requests.

   ```bash
   kubectl wait -n a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=aspnetapp --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   kubectl wait -n a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=aspnetapp --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   kubectl wait -n a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=aspnetapp-canary --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   ```

1. Check the status of your ingress resources. This helps you to confirm the AKS-managed internal load balancer is functioning.

   Traefik reads your ingress resource object configuration, updates its status, and creates a router to fulfill the new exposed workloads route.

   ```bash
   kubectl get IngressRoute aspnetapp-ingress -n a0042 --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   kubectl get IngressRoute aspnetapp-ingress -n a0042 --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   ```

   At this point, the route to the workload is established, SSL offloading is configured, and a network policy is in place to only allow Traefik to connect to your workload. Therefore, you should expect a `403` HTTP response if you attempt to connect to it directly.

> :book: The app team is happy to confirm to the BU0001 that their workload is now consuming less memory thanks to this update.

![The app team is doing some canary testing of their workload, and sees just a few requests are routed to the new version. The new ASP.NET workload app is reporting less memory usage.](images/canary-testing.gif)

### Next step

:arrow_forward: [End to End Validation](./10-validation.md)
