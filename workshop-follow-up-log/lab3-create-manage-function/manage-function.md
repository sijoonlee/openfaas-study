## Managin multiple functions
- create two function using 'append' in one yml
    ```
    faas-cli new --lang python3 first
    faas-cli new --lang python3 second --append=./first.yml
    mv first.yml example.yml
    ```
- example.yml
    ```
    version: 1.0
    provider:
    name: openfaas
    gateway: http://127.0.0.1:31112
    functions:
    first:
        lang: python3
        handler: ./first
        image: sijoonlee/first:latest

    second:
        lang: python3
        handler: ./second
        image: sijoonlee/second:latest
    ```
- build in parallel
    ```
    faas-cli build -f ./example.yml --parallel=2
    ```

- build / push only one function:
    ```
    faas-cli build -f ./example.yml --filter=second
    ```

## Custom Templates

### pulling custom templates
- If you have your own set of forked or custom templates, then you can pull them down for use with the CLI
- Here's an example of fetching a Python 3 template which uses Debian Linux
- Pull the template
    ```
    faas-cli template pull https://github.com/openfaas-incubator/python3-debian
    ```
- faas-cli new --list | grep python
    ```
    - python
    - python3
    - python3-debian
    ```
- These new templates are saved in your current working directory in ./templates/

### template store
- faas-cli template store list
- faas-cli template store list -v
- faas-cli template store describe node10-express
- faas-cli template store pull node10-express
- now you can use 
    ```
    faas-cli new --lang node10-express
    ```