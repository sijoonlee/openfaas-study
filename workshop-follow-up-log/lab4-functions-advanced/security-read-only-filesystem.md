## make the root filesystem read-only
- create function
    ```
    faas-cli new --lang python3 ingest-file --prefix=$OPENFAAS_PREFIX
    ```
- update ingest-file.yml - **readonly_root_filesystem: true**
    ```
    functions:
    ingest-file:
        lang: python3
        handler: ./ingest-file
        image: sijoonlee/ingest-file:latest
        readonly_root_filesystem: true
    ```
- update handler
    ```
    import os
    import time

    def handle(req):
        # Read the path or a default from environment variable
        path = os.getenv("save_path", "/home/app/")

        # generate a name using the current timestamp
        t = time.time()
        file_name = path + str(t)

        # write a file
        with open(file_name, "w") as f:
            f.write(req)
            f.close()

        return file_name
    ```
- build the function
    ```
    faas-cli up -f ingest-file.yml
    ```
- try below commands - should fail
    ```
    echo "Hello function" > message.txt
    cat message.txt | faas-cli invoke -f ingest-file.yml ingest-file
    ```
    ```
    Server returned unexpected status code: 500 - exit status 1
    Traceback (most recent call last):
    File "index.py", line 19, in <module>
        ret = handler.handle(st)
    File "/home/app/function/handler.py", line 13, in handle
        with open(file_name, "w") as f:
    OSError: [Errno 30] Read-only file system: '/home/app/1594210710.5858984'
    ```
- In order to write to a temporary area set the environment variable **save_path**
    ```
    functions:
    ingest-file:
        lang: python3
        handler: ./ingest-file
        image: alexellis2/ingest-file:latest
        readonly_root_filesystem: true
        environment:
            save_path: "/tmp/"
    ```