1. install OpenFaaS

```
curl -sLSf https://cli.openfaas.com | sudo sh
faas-cli help
faas-cli version
```

2. Configure a registry for Docker Hub
- login
    ```
    docker login
    ```
- Edit ~/.bashrc
    ```
    export OPENFAAS_PREFIX="" # Populate with your Docker Hub username
    ```

3. Setup a single-node cluster using Kubernetes and helm
- install helm (on /usr/local/bin/helm)
    ```
    curl -sSLf https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    ```
- creating two namspaces (namespace/openfaas, namespace/openfaas-fn)
    ```
    kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
    ```
    - namespaces.yml
        ```
        apiVersion: v1
        kind: Namespace
        metadata:
        name: openfaas
        annotations:
            linkerd.io/inject: enabled
            config.linkerd.io/skip-inbound-ports: "4222"
            config.linkerd.io/skip-outbound-ports: "4222"
        labels:
            role: openfaas-system
            access: openfaas-system
            istio-injection: enabled
        ---
        apiVersion: v1
        kind: Namespace
        metadata:
        name: openfaas-fn
        annotations:
            linkerd.io/inject: enabled
            config.linkerd.io/skip-inbound-ports: "4222"
            config.linkerd.io/skip-outbound-ports: "4222"
        labels:
            istio-injection: enabled
            role: openfaas-fn
        ```
- add the OpenFaaS helm chart
    ```
    helm repo add openfaas https://openfaas.github.io/faas-netes/
    ```
- Now decide how you want to expose the services and edit the **helm upgrade** command as required.
    ```
    To use NodePorts (default) pass no additional flags
    To use a LoadBalancer add --set serviceType=LoadBalancer
    To use an IngressController add --set ingress.enabled=true
    ```
- deploy
    ```
    helm repo update \
    && helm upgrade openfaas --install openfaas/openfaas \
        --namespace openfaas  \
        --set functionNamespace=openfaas-fn \
        --set generateBasicAuth=true 
    ```
    ```
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "openfaas" chart repository
    Update Complete. ⎈ Happy Helming!⎈ 
    Release "openfaas" does not exist. Installing it now.
    NAME: openfaas
    LAST DEPLOYED: Tue Jul  7 09:50:40 2020
    NAMESPACE: openfaas
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    To verify that openfaas has started, run:

    kubectl -n openfaas get deployments -l "release=openfaas, app=openfaas"
    To retrieve the admin password, run:

    echo $(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode)
```
- Retrieve the OpenFaaS credentials
    ```
    PASSWORD=$(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode) && \
echo "OpenFaaS admin password: $PASSWORD"
    ```
    ```
    OpenFaaS admin password: 9ksCH7Bou2S7
    ```
- Tuning cold-start: The concept of a cold-start in OpenFaaS only applies if you 
    A) use faas-idler and 
    B) set a specific function to scale to zero. 
    Otherwise there is not a cold-start, because at least one replica of your function remains available.
    - There are two ways to reduce the Kubernetes cold-start for a pre-pulled image, which is around 1-2 seconds.
        1. Don't set the function to scale down to zero, just set it a minimum availability i.e. 1/1 replicas
        2. Use async invocations via the /async-function/<name> route on the gateway, so that the latency is hidden from the caller
        3. Tune the readinessProbes to be aggressively low values. This will reduce the cold-start at the cost of more kubelet CPU usage
        - To achieve around 1s coldstart, set values.yaml:
        ```
        faasnetes:

        # redacted
        readinessProbe:
            initialDelaySeconds: 0
            timeoutSeconds: 1
            periodSeconds: 1
        livenessProbe:
            initialDelaySeconds: 0
            timeoutSeconds: 1
            periodSeconds: 1
        # redacted
        imagePullPolicy: "IfNotPresent"    # Image pull policy for deployed functions
        ```
- verify the installation
    1. fetch NodePort (or IP)
    ```
    kubectl get svc -n openfaas gateway-external -o wide
    ```
    ```
    NAME               TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE   SELECTOR
    gateway-external   NodePort   10.152.183.213   <none>        8080:31112/TCP   21m   app=gateway
    ```

    2. set it as an environment variable (Edit ~/.bashrc for later use in next the workshops)
    ```
    export OPENFAAS_URL=http://127.0.0.1:31112
    ```
    3. log in and check
    ```
    # This command logs in and saves a file to ~/.openfaas/config.yml
    echo -n $PASSWORD | faas-cli login -g $OPENFAAS_URL -u admin --password-stdin
    faas-cli version
    ```
    ```
    CLI:
    commit:  16f6eb9522cff9622b78cbe6450d60f8b3cd7ead
    version: 0.12.8

    Gateway
    uri:     http://127.0.0.1:31112
    version: 0.18.17
    sha:     18f6c720b50db7da5f9c410f9fd3369ed7aff379
    commit:  Extract a caching function_query type

    Provider
    name:          faas-netes
    orchestration: kubernetes
    version:       0.10.5 
    sha:           9be50543b372381a505e9e54a1356bb076c8f01f
    ```

4. check OpenFaaS gateway
- check the gateway is ready
    ```
    kubectl rollout status -n openfaas deploy/gateway
    ```
- list
    ```
    faas-cli list
    ```