1. open web ui
    ```
    address: http://127.0.0.1:31112/
    admin / 9ksCH7Bou2S7
    ```

2. deploy sample functions
    ```
    faas-cli deploy -f https://raw.githubusercontent.com/openfaas/faas/master/stack.yml
    ```
    - stack.yml
    ```
    provider:
        name: openfaas
        gateway: http://127.0.0.1:8080  # can be a remote server

    functions:
        # Pass a username as an argument to find how many images user has pushed to Docker Hub.
        hubstats:
            lang: dockerfile
            image: functions/hubstats:latest
            skip_build: true

        # Node.js gives OS info about the node (Host)
        nodeinfo:
            lang: dockerfile
            image: functions/nodeinfo:latest
            skip_build: true

        # Uses `cat` to echo back response, fastest function to execute.
        echoit:
            lang: dockerfile
            image: functions/alpine:latest
            fprocess: "cat"
            skip_build: true

        # Counts words in request with `wc` utility
        wordcount:
            lang: dockerfile
            image: functions/alpine:latest
            fprocess: "wc"
            skip_build: true

        # Calculates base64 representation of request body.
        base64:
            lang: dockerfile
            image: functions/alpine:latest
            fprocess: "base64"
            skip_build: true

        # Converts body in (markdown format) -> (html)
        markdown:
            lang: dockerfile
            image: functions/markdown-render:latest
            skip_build: true
    ```

3. list the deployed functions
- faas-cli list
    ```
    Function                      	Invocations    	Replicas
    base64                        	0              	1    
    echoit                        	0              	1    
    hubstats                      	0              	1    
    markdown                      	0              	1    
    nodeinfo                      	0              	1    
    wordcount                     	0              	1   
    ```
- faas-cli list --verbose
- faas-cli list -v
    ```
    Function                      	Image                                   	Invocations    	Replicas
    base64                        	functions/alpine:latest                 	0              	1    
    echoit                        	functions/alpine:latest                 	0              	1    
    hubstats                      	functions/hubstats:latest               	0              	1    
    markdown                      	functions/markdown-render:latest        	0              	1    
    nodeinfo                      	functions/nodeinfo:latest               	0              	1    
    wordcount                     	functions/alpine:latest                 	0              	1   
    ```

4. Invoke function
- faas-cli invoke markdown
    ```
    Reading from STDIN - hit (Control + D) to stop.
    **aa**<p><strong>aa</strong></p>
    ```
- echo Hi | faas-cli invoke markdown
    ```
    <p>Hi</p>
    ```
- uname -a | faas-cli invoke markdown
    ```
    <p>Linux ubuntu 5.3.0-62-generic #56-Ubuntu SMP Tue Jun 23 11:20:52 UTC 2020 x86<em>64 x86</em>64 x86_64 GNU/Linux</p>
    ```
- cat lab2.md | faas-cli invoke markdown
    ```
    <ol>
    <li><p>open web ui
    <code>
    ```

5. Monitoring dashboard
OpenFaaS tracks metrics on your functions automatically using Prometheus. The metrics can be turned into a useful dashboard with free and Open Source tools like Grafana.

- Run Grafana in OpenFaaS Kubernetes namespace
    ```
    kubectl -n openfaas run \
    --image=stefanprodan/faas-grafana:4.6.3 \
    --port=3000 \
    grafana
    ```
- Expose Grafana with a NodePort
    ```
    kubectl -n openfaas expose pod grafana-69f9dbb4cd-cr5ts \
    --type=NodePort \
    --name=grafana
    ```
- Find Grafana node port address
    ```
    GRAFANA_PORT=$(kubectl -n openfaas get svc grafana -o jsonpath="{.spec.ports[0].nodePort}")
    GRAFANA_URL=http://127.0.0.1:$GRAFANA_PORT/dashboard/db/openfaas
    echo $GRAFANA_URL
    ```
    ```
    http://127.0.0.1:30157/dashboard/db/openfaas
    ```
- log in to Grafana
    ```
    admin / admin
    ```