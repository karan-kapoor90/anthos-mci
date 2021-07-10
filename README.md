# Objective

The objective is create a service mesh using multiple kubernetes clusters distributed around the globe, to host services for customers to consume. Here are a couple of things that should happen:
- Users coming from India should be redirected to the pods running closest to them. Users coming from Australia should be redirected to the clusters in Australia.
- Routing of services should be defined using a single ingress with a single global IP address
- Service Discovery and inter-cluster routing should be possible within the 2 clusters internally, without going over the public internet. 

You need to do 2 things
1. Setup multi-cluster mesh using ASM
2. Setup multi-cluster ingress that uses a Global LB for load balancing and can do intelligent routing

## Assumptions

- Both the clusters are in the same GCP project
- Both the clusters are in the same GCP VPC
- Only the app-tier is considered for this demonstration
- This is not a production grade deployment by any means. This is only for experimenting and experiencing some of the capabilities offered by Anthos Service Mesh on Google Cloud. 

# Multi-cluster mesh setup using ASM

References: https://cloud.google.com/service-mesh/docs/scripted-install/asm-onboarding then https://cloud.google.com/service-mesh/docs/scripted-install/gke-project-cluster-setup then https://cloud.google.com/service-mesh/docs/scripted-install/gke-install 
References: https://cloud.google.com/service-mesh/docs/gke-install-multi-cluster 

- Create 2 clusters in 2 different regions. Do pass the `--enable-alias-ip`
    ```bash
    gcloud container clusters create cluster1 --region asia-southeast1 --enable-ip-alias
    gcloud container clusters create cluster2 --region australia-southeast1 --enable-ip-alias
    ```
- Install ASM on both the clusters 
- Configure endpoint discovery between clusters - https://cloud.google.com/service-mesh/docs/gke-install-multi-cluster#verify_your_deployment
    - Switch context into first cluster and apply the output of `istioctl x create-remote-secret` into the second cluster 
    - Switch context into second cluster and apply the output of `istioctl x create-remote-secret` into the first cluster - `istioctl x create-remote-secret | kubectl apply -f - --context <context-name-for-first-cluster>`
- Verify istio configuration
    - enable sidecar injection
    - Deploy svc
    - deploy v1 and v2 on cluster1 and 2 respectively
    - deploy sleep containers
    - verify load balancing b/w pods on the 2 clusters using the sleep containers
    - Delete the sample namespace on both the clusters


# Multi-cluster ingress setup


Reference: https://docs.google.com/document/d/1VDlGHlYq8uw4JTsq-iMjrLmptQ6gMgtcNHwWFlYbrjg/edit?resourcekey=0-wdodkiXzWXbCrSjnwzLS7A

```bash
export PROJECT=$(gcloud config get-value project)
```

- Create a global static Ip address for the Global LB (HTTP LB)

    ```bash
    gcloud compute addresses create glb-ip --global

    # Note the external IP address created here. If you're not able to see the IP, wait for a couple of minutes before trying
    gcloud compute addresses list --global 

    ```

- Create a DNS entry that links the URL that users will use to access your application, with the GLB-IP.

    ```bash
    cat <<EOF > multicluster/dns-spec.yaml
    swagger: "2.0"
    info:
    description: "Cloud Endpoints DNS"
    title: "Cloud Endpoints DNS"
    version: "1.0.0"
    paths: {}
    host: "frontend.endpoints.${PROJECT}.cloud.goog"
    x-google-endpoints:
    - name: "frontend.endpoints.${PROJECT}.cloud.goog"
    target: "${GCLB_IP}"
    EOF

    gcloud endpoints services deploy multicluster/dns-spec.yaml
    ```


Reference: https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress-setup#requirements_for 

- The cluster has to be registered as a member of the fleet of the asm enabled clusters. This can eithe be done from the anthos UI console, or using the command 

    ```bash
    gcloud container hub memberships register gke-eu \
        --project=project-id \
        --gke-uri=uri \
        --service-account-key-file=service-account-key-path
    ```

Once done, we have to select one of the clusters as the config cluster. This will house the MCI and MCS resources. Only 1 cluster in a fleet can be an active config cluster

> Note: For me the whole path passed as the config-membership param didn't work, so I passed cluster as the value to the config-membership flag.

    ```bash
    gcloud alpha container hub ingress enable \
    --config-membership=projects/project_id/locations/global/memberships/gke-us   #Command as per documentation

    gcloud alpha container hub ingress enable \
    --config-membership=cluster1  # The actual command I used.
    ```

- Confirm cluster registration as the config cluster

    ```bash
    gcloud alpha container hub ingress describe
    ```

If all is running well here, now to create an actual ingress resource

Reference: https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress 

- Create a namespace for the `zoneprinter` app and register it for istio sidecar injection

    ```bash
    # get the revision label value from the istiod service in the istio-system
    kubectl -n istio-system get pods -l app=istiod

    # output
    NAME                                 READY   STATUS    RESTARTS   AGE
    istiod-asm-1102-3-76b8d69bf9-9mft7   1/1     Running   0          17h
    istiod-asm-1102-3-76b8d69bf9-gvsw9   1/1     Running   0          17h

    # Note the 'asm-1102-3' from the pod name

    # on both the clusters, run these steps and insert the version from the pod name into the label
    kubectl create ns zoneprinter
    kubectl label ns zoneprinter istio-injection- istio.io/rev=asm-1102-3 --overwrite
    ```

- Deploy the zoneprinter application into both the clusters by deploying the `resources/deploy.yaml`
- Deploy the `MultiClusterService` and the `MultiClusterIngress` into the config cluster - `cluster1` 
- Validate the MCI (MultiClusterIngress) deployment status using `kubectl describe mci zone-ingress -n zoneprinter` 
- Retreive the Public IP address of the HTTP LB created using the MCI - `kubectl get mci zone-ingress -o=jsonpath={.status.VIP}`. This VIP will be the same IP reservation we made earlier.
- The site will be available at the FQDN entry we made in the registry earlier. This can be retrieved using `gcloud endpoints services list --format="value(serviceName)"`


Test the services from different regions using your local broswer as well as https://geotargetly.com/geo-browse 


# Multi-cluster service discovery

Create all sorts of routing rules and ingress configurations using - https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress#features 