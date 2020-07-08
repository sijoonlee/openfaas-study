## log in
    ```
    PASSWORD=$(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode) && echo -n $PASSWORD | faas-cli login -g $OPENFAAS_URL -u admin --password-stdin
    ```

## Creating a new function

### Scaffold or generate a new function
1. use template
    ```
    faas-cli template pull
    ```
    - this will generate folder 'template'
    - Alternatively create a folder containing a Dockerfile, and pick **Dockerfile** lang type in YAML file

2. Hello-world function in Python
- generate function files
    ```
    faas-cli new --lang python3 hello-openfaas --prefix=$OPENFAAS_PREFIX
    ```
    - *OPENFAAS_PREFIX* is docker username
    - The --prefix parameter will update image: value in hello-openfaas.yml with a prefix which should be your Docker Hub account.
- hello-openfaas.yml
    ```
    version: 1.0
    provider:
        name: openfaas
        gateway: http://127.0.0.1:31112
    functions:
        hello-openfaas:
            lang: python3
            handler: ./hello-openfaas                   // should be directory
            image: sijoonlee/hello-openfaas:latest      // note that docker account is used as prefix
    ```
- edit handler.py - return "Hello World"
- sudo -E faas-cli up -f hello-openfaas.yml
    ```
    Note: Please make sure that you have logged in to docker registry with docker login command before running this command.
    Note: faas-cli up command combines build, push and deploy commands of faas-cli in a single command.
    ```
    ```
    Deployed. 202 Accepted.
    URL: http://127.0.0.1:31112/function/hello-openfaas.openfaas-fn

    ```
- The function will always get a route, for example:
    ```
    $OPENFAAS_URL/function/<function_name>
    $OPENFAAS_URL/function/hello-openfaas
    ```

- Invoke function
    - Functions can be invoked via a GET or POST method only
    - faas-cli invoke hello-openfaas

3. astronaut-finder
- create function
    ```
    faas-cli new --lang python3 astronaut-finder --prefix=$OPENFAAS_PREFIX
    ```
- list pip modules **requests** @./astronaut-finder/requirements.txt
- edit handler.py
    ```
    import requests
    import random

    def handle(req):
        r = requests.get("http://api.open-notify.org/astros.json")
        result = r.json()
        index = random.randint(0, len(result["people"])-1)
        name = result["people"][index]["name"]

        return "%s is in space" % (name)
    ```
- build/push/deploy
    ```
    sudo -E faas-cli build -f ./astronaut-finder.yml
    sudo -E faas-cli push -f ./astronaut-finder.yml
    sudo -E faas-cli deploy -f ./astronaut-finder.yml
    ```
- invoke
    ```
    echo | faas-cli invoke astronaut-finder
    ```
- log: kubectl logs deployment/astronaut-finder -n openfaas-fn
    ```
    2020/07/07 20:22:42 Version: 0.18.1	SHA: b46be5a4d9d9d55da9c4b1e50d86346e0afccf2d
    2020/07/07 20:22:42 Timeouts: read: 5s, write: 5s hard: 0s.
    2020/07/07 20:22:42 Listening on port: 8080
    2020/07/07 20:22:42 Writing lock-file to: /tmp/.lock
    2020/07/07 20:22:42 Metrics listening on port: 8081
    2020/07/07 20:22:57 Forking fprocess.
    2020/07/07 20:22:58 Wrote 26 Bytes - Duration: 0.635436 seconds
    2020/07/07 20:25:01 Forking fprocess.
    2020/07/07 20:25:01 Wrote 30 Bytes - Duration: 0.598291 seconds
    ```
- Add to yaml **write_debug: true**
    ```
    astronaut-finder:
        lang: python3
        handler: ./astronaut-finder
        image: sijoonlee/astronaut-finder:latest
        environment:
            write_debug: true
    ```
- log: kubectl logs deployment/astronaut-finder -n openfaas-fn
    ```
    020/07/07 20:27:26 Version: 0.18.1	SHA: b46be5a4d9d9d55da9c4b1e50d86346e0afccf2d
    2020/07/07 20:27:26 Timeouts: read: 5s, write: 5s hard: 0s.
    2020/07/07 20:27:26 Listening on port: 8080
    2020/07/07 20:27:26 Writing lock-file to: /tmp/.lock
    2020/07/07 20:27:26 Metrics listening on port: 8081
    2020/07/07 20:28:21 Forking fprocess.
    2020/07/07 20:28:21 Query  
    2020/07/07 20:28:21 Path  /
    2020/07/07 20:28:21 Duration: 0.653806 seconds
    Bob Behnken is in space
    ```

4. Binary file support
- OpenFaaS supports any binary including bash commands, C#, VB or PowerShell
- faas-cli new --lang dockerfile sorter --prefix=$OPENFAAS_PREFIX
- Edit sorter/Dockerfile to update the line
    ```
    ENV fprocess="sort"
    ```
    *sort*: built-in bash command
- faas-cli up -f sorter.yml
- invoke
    ```
    $ echo -n '
    elephant
    zebra
    horse
    aardvark
    monkey'| faas-cli invoke sorter

    aardvark
    elephant
    horse
    monkey
    zebra
    ```