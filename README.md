# f5-nginx-ingress-on-gcp

This is a step-by-step lab that shows how to deploy **NGINX Plus** as an *ingress controller* for your kubernetes cluster - more specifically a GKE cluster. This lab also explore the use of **NGIX App Protect** (WAF) to protect your workloads running on a kubernetes cluster. All events logs from **NGINX App Protect** are sent to the **Cloud Operations Logging** service.

## Overview

This is high-level view about he steps performed in this lab:

1. Create a GKE cluster ;
2. Build a **NGINX Plus Ingress Controller** container image (with **NGINX App Protect** enabled) and push it to our *Google Container Registry* private repo ; 
3. Use **Helm** to install the **NGINX Ingress Controller** chart using our previously built container image ; 
4. Deploy the demo **Cafe** application (composed by two services, *coffee* and *tea*)
5. Build a container image for our custom syslog service which will use the *google-fluentd* agent to parse and send the NGINX App Protect event logs to the *Cloud Operations Logging* service ; 
6. Deploy our custom syslog service (*Deployment* and *Service* resources) ;
7. Generate a certificate/key pair ;
8. Create a **Secret** resource (which will be used by our *Ingress* resource);
9. Create a NAP **APPolicy** resource (which will be used by our *Ingress* resource);
10. Create a NAP **APLogConf** resource (which will be used by our *Ingress* resource);
11. Create a **Ingress** resource configured to use our *NGINX Ingress Controller* and *NGINX App Protect*. The resource is also configured to send all NAP event logs to our custom syslog service (which will forward them to *Cloud Operations Logging*) ; 
12. Test the application and launch some attacks ; 
13. Look the NAP event logs on *Cloud Operations Logging* and run some queries ; 

## F5 NGINX Plus as Ingress Controller with NGINX App Protect 

1. Define some environment variables:

    ```
    export PROJECTNAME="f5-nginx-lab-001"
    export REGION="us-central1"
    export ZONE="us-central1-a"
    ```

2. Create a GCP project: 

    ```
    gcloud projects create $PROJECTNAME --name="My F5 NGINX Lab" 
    ```

3. Configure the newly created project as the default project:

    ```
    gcloud config set project $PROJECTNAME
    ```

4. Get the billing account ID which will be used by this project: 

    ```
    gcloud alpha billing accounts list
    ```

5. Link the newly created project with your billing account :

    ```
    gcloud alpha billing projects link $PROJECTNAME --billing-account XXXXXX-XXXXXX-XXXXXX
    ```

6. Get your public IP and save it in an environment variable (this public IP will be used to restrict the access to your lab environment): 

    ```
    export MYIP=$(curl api.ipify.org)
    ```

7. Enable some GCP APIs that will be used later on:

    ```
    gcloud services enable compute.googleapis.com
    gcloud services enable container.googleapis.com
    gcloud services enable cloudbuild.googleapis.com

8. Create a GKE cluster:

    ```
    gcloud container clusters create f5-nginx-gke-cluster --project=$PROJECTNAME --zone=$ZONE 
    ```
9. Configure the *docker* command to use the *Google Container Registry* (which will be used to push the NGINX Ingress Controller container image):

    ```
    gcloud auth configure-docker
    ```

10. Clone the **NGINX Ingress Controller** repo:

    ```
    git clone https://github.com/nginxinc/kubernetes-ingress/
    cd kubernetes-ingress
    git checkout v2.1.1
    ```

11. Copy the certificate and key of our license to the root of **NGINX Ingress Controller** project:

    ```
    cp ../nginx-repo/nginx-repo.crt .
    cp ../nginx-repo/nginx-repo.key .
    ```

12. Build the **NGINX Plus Ingress Controller** container image:

    ```
    make debian-image-nap-plus PREFIX=gcr.io/$PROJECTNAME/nginx-plus-ingress TARGET=download
    ```

13. Push the **NGINX Plus Ingress Controller** container image to your private *Google Container Registry* repo: 

    ```
    make push PREFIX=gcr.io/$PROJECTNAME/nginx-plus-ingress && cd ..
    ```

14. Check the pushed image:

    ```
    gcloud container images list --repository=gcr.io/$PROJECTNAME
    NAME
    gcr.io/f5-nginx-lab-001/nginx-plus-ingress
    ```

15. Get the image tag (will be used when installing the NGINX Ingress Controller):

    ```
    export IMAGETAG=`gcloud container images list-tags gcr.io/$PROJECTNAME/nginx-plus-ingress --format="value(tags[0])"`
    ```

15. Configure *kubectl* command to use your GKE cluster:  

    ```
    gcloud container clusters get-credentials f5-nginx-gke-cluster
    ```

16. Create the namespace **nginx-ingress**:

    ```
    kubectl create namespace nginx-ingress
    ```

17. Add the **Helm** repository:

    ```
    helm repo add nginx-stable https://helm.nginx.com/stable
    helm repo update
    ```

18. Install **NGINX Ingress Controller** chart (using our previously build NGINX Plus Ingress Controller container image):

    ```
    helm install nginx-ingress nginx-stable/nginx-ingress --namespace nginx-ingress \
     --set controller.image.repository=gcr.io/$PROJECTNAME/nginx-plus-ingress \
     --set controller.nginxplus=true \
     --set controller.replicaCount=3 \
     --set controller.appprotect.enable=true \
     --set controller.image.tag=$IMAGETAG \
     --set controller.ingressClass="nginx"
    ```

20. Check the **NGINX Ingress Controller** PODs:

    ```
    kubectl get pods -n nginx-ingress
    NAME                                           READY   STATUS    RESTARTS   AGE
    nginx-ingress-nginx-ingress-858d9d86c6-4ctcg   1/1     Running   0          26s
    nginx-ingress-nginx-ingress-858d9d86c6-dhbsv   1/1     Running   0          26s
    nginx-ingress-nginx-ingress-858d9d86c6-hwplx   1/1     Running   0          26s
    ```
    
21. Find the public IP address of your **NGINX Plus Ingress Controller** Service:

    ```
    kubectl get svc nginx-ingress-nginx-ingress -n nginx-ingress
    ```

    **Note:** Run the above command until you see the public IP address (this can take some time).

22. Get the public IP address of your **NGINX Plus Ingress Controller** Service (and put this on a environment variable):

    ```
    export NGINX_IC_IP=`kubectl get svc nginx-ingress-nginx-ingress -n nginx-ingress -o json | jq -r '.status.loadBalancer.ingress[0].ip'`
    ```

23. Deploy the **Cafe** application (composed by two services, *coffee* and *tea*):

    ```
    kubectl apply -f k8s_manifests/cafe.yaml 
    ```

24. Check the resources which were created (*Deployment*, *Pod*, *Service*): 

    ```
    kubectl get deployment
    NAME     READY   UP-TO-DATE   AVAILABLE   AGE
    coffee   2/2     2            2           38s
    tea      3/3     3            3           37s

    kubectl get pods
    NAME                      READY   STATUS    RESTARTS   AGE
    coffee-55bf956c8d-9htn2   1/1     Running   0          37s
    coffee-55bf956c8d-ldblw   1/1     Running   0          37s
    tea-84d7c495b4-4wffc      1/1     Running   0          37s
    tea-84d7c495b4-m46dv      1/1     Running   0          37s
    tea-84d7c495b4-pxc2s      1/1     Running   0          37s
    
    kubectl get svc
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    coffee-svc   ClusterIP   10.116.1.213    <none>        80/TCP    39s
    kubernetes   ClusterIP   10.116.0.1      <none>        443/TCP   17m
    tea-svc      ClusterIP   10.116.11.158   <none>        80/TCP    38s
    ```

25. Use *Cloud Build* to build and push (to our GCR private repo) the **syslog-cloud-logging** container image (this is a custom image which runs the *google-fluentd* agent and will be responsible for receive the NGINX App Protect event logs, parse them and send them to the *Cloud Operations Logging* service):

    ```
    sed "s/PROJECTNAME/$PROJECTNAME/" syslog-cloud-logging/cloudbuild.original.yaml > syslog-cloud-logging/cloudbuild.yaml
    gcloud builds submit --config syslog-cloud-logging/cloudbuild.yaml
    ```

26. Check the pushed image:

    ```
    gcloud container images list --repository=gcr.io/$PROJECTNAME
    NAME
    gcr.io/f5-nginx-lab-001/nginx-plus-ingress
    gcr.io/f5-nginx-lab-001/syslog-cloud-logging
    ```

27. Deploy the **syslog** service (using our previously built container image):

    ```
    sed "s/PROJECTNAME/$PROJECTNAME/" k8s_manifests/syslog.original.yaml > k8s_manifests/syslog.yaml
    kubectl apply -f k8s_manifests/syslog.yaml
    ```
    **Note**: The *Ingress* resource will be configured to send all the NAP event logs to this **syslog** service which will then forward these event logs to the *Cloud Operations Logging* service.

28. Create a certificate/key pair:

    ```
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=cafe.example.com"
    ```

29. Get the certificate and key and put them in some environment variables:

    ```
    export TLS_CRT=`cat tls.crt | base64 -w 0`
    export TLS_KEY=`cat tls.key | base64 -w 0`
    ```

30. Create the *Secret* resource file:

    ```
    sed -e "s/TLS_CRT/$TLS_CRT/g" -e "s/TLS_KEY/$TLS_KEY/g" k8s_manifests/cafe-secret.original.yaml > k8s_manifests/cafe-secret.yaml
    ```

31. Check the *Secret* resource file:

    ```
    less k8s_manifests/cafe-secret.yaml
    ```

32. Create the **Secret** resource (which will be used when creating the *Ingress* resource ):

    ```
    kubectl apply -f k8s_manifests/cafe-secret.yaml
    ```

33. Create the NAP **APPolicy** (a.k.a WAF Policy) resource:

    ```
    kubectl apply -f k8s_manifests/cafe-nap-appolicy.yaml
    ```

34. Create the NAP **APLogConf** resource:

    ```
    kubectl apply -f k8s_manifests/cafe-nap-aplogconf.yaml
    ```

35. Create the **Ingress** resource:

    ```
    kubectl apply -f k8s_manifests/cafe-ingress.yaml
    ```

36. Test the application:

    ```
    curl -k -s https://$NGINX_IC_IP/coffee -H "Host: cafe.example.com"
    Server address: 10.112.1.6:80
    Server name: coffee-55bf956c8d-9htn2
    Date: 28/Feb/2022:01:48:51 +0000
    URI: /coffee
    Request ID: 545745030d4719450a1b1886d2d383ea

    curl -k -s https://$NGINX_IC_IP/tea -H "Host: cafe.example.com"
    Server address: 10.112.1.8:80
    Server name: tea-84d7c495b4-pxc2s
    Date: 28/Feb/2022:01:48:53 +0000
    URI: /tea
    Request ID: c0fc57752f81d729d7a383f55667b37a
    ```

37. Launch some attacks:

    ```
    curl -k -s "https://$NGINX_IC_IP/coffee?name=<script>alert('xss');</alert>" -H "Host: cafe.example.com"
    <html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 14517474674167111961<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>

    curl -k -s "https://$NGINX_IC_IP/coffee?name=test'%20OR%201=1%20#" -H "Host: cafe.example.com"
    <html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 14517474674167112471<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
    
    curl -k -s "https://$NGINX_IC_IP/coffee?name=test%20|%20cat%20/etc/passwd" -H "Host: cafe.example.com"
    <html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 7003483534934422482<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
    ```

38. Run some queries on *Google Cloud Operations Logging*:

    - Search for all NAP event logs ( ```logName=~"nginx-app-protect"``` )
    - Search for all NAP event logs with an *outcome* of *REJECTED* ( ```logName=~"nginx-app-protect" jsonPayload.outcome="REJECTED"```
    - Search for all NAP event logs with a *request status* of *blocked* ( ```logName=~"nginx-app-protect" jsonPayload.request_status="blocked"```)
    - Search for all NAP event logs from a specific NAP policy ( ```logName=~"nginx-app-protect" jsonPayload.policy_name="cafe_nap_appolicy"```)
    - Search for all NAP event logs with a *violation rating* greater than or equal to *4* (```logName=~"nginx-app-protect" jsonPayload.violation_rating>=4```)
    - Search for a NAP event log with a specific *support id* ( ```logName=~"nginx-app-protect" jsonPayload.support_id="7003483534934422482"```) 

    ![Cloud Logging - NGINX App Protect 1](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-01.png)
   
    ![Cloud Logging - NGINX App Protect 2](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-02.png)
   
    ![Cloud Logging - NGINX App Protect 3](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-03.png)
   
    ![Cloud Logging - NGINX App Protect 4](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-04.png)
   
    ![Cloud Logging - NGINX App Protect 5](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-05.png)
   
    ![Cloud Logging - NGINX App Protect 6](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-06.png)
   
    ![Cloud Logging - NGINX App Protect 7](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-07.png)

    ![Cloud Logging - NGINX App Protect 8](https://github.com/pedrorouremalta/f5-nginx-ingress-on-gcp/blob/master/images/nap-08.png)

## Cleaning up the lab environment (step-by-step)

1. Delete the kubernetes resources (*optional, you can just delete the entire GKE cluster*):

    ```
    kubectl delete ingress cafe-ingress
    kubectl delete svc coffee-svc syslog-svc tea-svc
    kubectl delete deploy coffee syslog tea
    kubectl delete appolicy cafe-nap-appolicy
    kubectl delete aplogconf cafe-nap-aplogconf
    ```

2. Uninstall **NGINX Ingress Controller** chart (*optional, you can just delete the entire GKE cluster*):

    ```
    helm uninstall nginx-ingress -n nginx-ingress
    ```

3. Delete the **nginx-ingress** namespace (*optional, you can just delete the entire GKE cluster*):

    ```
    kubectl delete namespace nginx-ingress
    ```

4. Delete the GKE cluster:

    ```
    gcloud container clusters delete f5-nginx-gke-cluster --quiet
    ```

5. Delete the container images previously built:

    ```
    gcloud container images delete gcr.io/$PROJECTNAME/syslog-cloud-logging:latest --quiet
    gcloud container images delete gcr.io/$PROJECTNAME/nginx-plus-ingress:$IMAGETAG --quiet
    ```

6. Delete some temporary files:

    ```
    rm tls.key tls.crt
    rm k8s_manifests/cafe-secret.yaml
    rm k8s_manifests/syslog.yaml
    rm -rf kubernetes-ingress/
    rm syslog-cloud-logging/cloudbuild.yaml
    ```