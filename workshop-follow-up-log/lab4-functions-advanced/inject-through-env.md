## Inject configuration through environment variables

1. At deployment time
- Add to yaml **write_debug: true**
    ```
    astronaut-finder:
        lang: python3
        handler: ./astronaut-finder
        image: sijoonlee/astronaut-finder:latest
        environment:
            write_debug: true
    ```
2. HTTP context - querystring / headers
- use querystring and HTTP headers
- Deploy a function that prints environmental variables using a built-in BusyBox command
    - deploy
        ```
        faas-cli deploy --name env --fprocess="env" --image="functions/alpine:latest
        ```
    - Invoke the function with a querystring (faas-cli)
        ```
        $ echo "" | faas-cli invoke env --query workshop=1
        ```
        ```
        PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        HOSTNAME=05e8db360c5a
        fprocess=env
        HOME=/root
        Http_Connection=close
        Http_Content_Type=text/plain
        Http_X_Call_Id=cdbed396-a20a-43fe-9123-1d5a122c976d
        Http_X_Forwarded_For=10.255.0.2
        Http_X_Start_Time=1519729562486546741
        Http_User_Agent=Go-http-client/1.1
        Http_Accept_Encoding=gzip
        Http_Method=POST
        Http_ContentLength=-1
        Http_Path=/
        ...
        Http_Query=workshop=1
        ...
        ```
    - Invoke the function with curl GET querystring
        ```
        $ curl -X GET $OPENFAAS_URL/function/env/some/path -d ""
        ```
        ```
        PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        HOSTNAME=fae2ac4b75f9
        fprocess=env
        HOME=/root
        Http_X_Forwarded_Host=127.0.0.1:8080
        Http_X_Start_Time=1539370471902481800
        Http_Accept_Encoding=gzip
        Http_User_Agent=curl/7.54.0
        Http_Accept=*/*
        Http_X_Forwarded_For=10.255.0.2:60460
        Http_X_Call_Id=bb86b4fb-641b-463d-ae45-af68c1aa0d42
        Http_Method=GET
        Http_ContentLength=0
        ...
        Http_Path=/some/path
        ...
        ```
    - Invoke it with a header
        ```
        $ curl $OPENFAAS_URL/function/env --header "X-Output-Mode: json" -d ""
        ```
        ```
        PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        HOSTNAME=05e8db360c5a
        fprocess=env
        HOME=/root
        Http_X_Call_Id=8e597bcf-614f-4ca5-8f2e-f345d660db5e
        Http_X_Forwarded_For=10.255.0.2
        Http_X_Start_Time=1519729577415481886
        Http_Accept=*/*
        Http_Accept_Encoding=gzip
        Http_Connection=close
        Http_User_Agent=curl/7.55.1
        Http_Method=GET
        Http_ContentLength=0
        Http_Path=/
        ...
        Http_X_Output_Mode=json
        ```
    - to get "Http_Path", use os.getenv("Http_Path") if Python