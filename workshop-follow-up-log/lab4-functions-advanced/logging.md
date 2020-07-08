## logging
- The OpenFaaS watchdog operates by passing in the HTTP request and reading an HTTP response via the standard I/O streams stdin and stdout
- create function
    ```
    faas-cli new --lang python3 logging-demo --prefix=$OPENFAAS_PREFIX
    ```
- update handler
    ```
    import sys
    import json

    def handle(req):
        sys.stderr.write("This should be an error message.\n")
        return json.dumps({"Hello": "OpenFaaS"})
    ```
- deploy
    ```
    faas-cli up -f logging-demo.yml
    ```
- invoke
    ```
    echo | faas-cli invoke logging-demo
    ```
    ```
    This should be an error message.
    {"Hello": "OpenFaaS"}
    ```
- update logging-demo.yml
    ```
    functions:
    logging-demo:
        lang: python3
        handler: ./logging-demo
        image: sijoonlee/logging-demo:latest
        environment:
        combine_output: false
    ```
- deploy
    ```
    faas-cli up -f logging-demo.yml
    ```
- invoke
    ```
    echo | faas-cli invoke logging-demo
    ```
    ```
    {"Hello": "OpenFaaS"}
    ```
- check container log
    ```
    kubectl logs logging-demo-6f4f6795c7-qd7qh --namespace=openfaas-fn
    ```
    ```
    2020/07/08 12:53:23 Version: 0.18.1	SHA: b46be5a4d9d9d55da9c4b1e50d86346e0afccf2d
    2020/07/08 12:53:23 Timeouts: read: 5s, write: 5s hard: 0s.
    2020/07/08 12:53:23 Listening on port: 8080
    2020/07/08 12:53:23 Writing lock-file to: /tmp/.lock
    2020/07/08 12:53:23 Metrics listening on port: 8081
    2020/07/08 12:53:35 Forking fprocess.
    2020/07/08 12:53:35 stderr: This should be an error message.
    2020/07/08 12:53:35 Wrote 22 Bytes - Duration: 0.077041 seconds
    ```