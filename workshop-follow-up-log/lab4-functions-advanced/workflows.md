## Workflows
- There will be situations where it will be useful to take the output of one function and use it as an input to another. 
- This is achievable both client-side and via the API Gateway.

### Chaining functions on the client-side
- You can pipe the result of one function into another using curl, the faas-cli or some of your own code
- Example
    - Deploy the NodeInfo function from the Function Store
    - Then push the output from NodeInfo through the Markdown converter
    ```
    $ echo -n "" | faas-cli invoke nodeinfo | faas-cli invoke markdown
    <p>Hostname: 64767782518c</p>

    <p>Platform: linux
    Arch: x64
    CPU count: 4
    Uptime: 1121466</p>
    ```

### Call one function from another
- The easiest way to call one function from another is make a call over HTTP via the OpenFaaS API Gateway
- This call does not need to know the external domain name or IP address
- It can simply refer to the API Gateway as gateway through a DNS entry
- In Lab 3 we introduced the requests module and used it to call a remote API to get the name of an astronaut aboard the ISS. We can use the same technique to call another function deployed on OpenFaaS.
- The default address for the gateway on Docker Swarm is http://gateway:8080
- The default address for the gateway on Kubernetes is http://gateway.openfaas:8080
- port 8080 is different from the external gateway's port
- get SentimentAnalysis function from store
    ```
    faas-cli store deploy SentimentAnalysis
    ```
    ```
    echo -n "California is great, it's always sunny there." | faas-cli invoke sentimentanalysis
    {"polarity": 0.8, "sentence_count": 1, "subjectivity": 0.75}
    ```
- create new function
    ```
    faas-cli new --lang python3 workflow-demo --prefix=$OPENFAAS_PREFIX
    ```
- add **requests** to requirements.txt file
- add below to workflow-demo.yml
    ```
    environment:
      gateway_hostname: gateway.openfaas
    ```
- update handler
    ```
    import os
    import requests
    import sys

    def handle(req):
        """handle a request to the function
        Args:
            req (str): request body
        """

        gateway_hostname = os.getenv("gateway_hostname", "gateway") # uses a default of "gateway" for when "gateway_hostname" is not set

        test_sentence = req

        r = requests.get("http://" + gateway_hostname + ":8080/function/sentimentanalysis", data= test_sentence)

        if r.status_code != 200:
            sys.exit("Error with sentimentanalysis, expected: %d, got: %d\n" % (200, r.status_code))

        result = r.json()
        if result["polarity"] > 0.45:
            return "That was probably positive"
        else:
            return "That was neutral or negative"
    ```
- deploy
    ```
    sudo faas-cli build -f workflow-demo.yml
    sudo faas-cli push -f workflow-demo.yml
    sudo -E faas-cli deploy -f workflow-demo.yml
    ```
- invoke
    ```
    echo -n "California is great, it's always sunny there." | faas-cli invoke workflow-demo
    ```
    ```
    That was probably positive
    ```