## set up a tunnel with ngrok
- to receive incoming webhooks from GitHub
```
kubectl -n openfaas run \
--image=alexellis2/ngrok-admin \
--port=4040 \
ngrok -- http gateway:8080
```
```
kubectl -n openfaas expose deployment ngrok \
--type=NodePort \
--name=ngrok
```
```
kubectl port-forward deployment/ngrok 4040:4040 -n openfaas
```
```
- Use the built-in UI of ngrok at http://127.0.0.1:4040 to find your HTTP URL. 
- You will be given a URL that you can access over the Internet, it will connect directly to your OpenFaaS API Gateway.

## Log into the gateway with the ngrok address
- set env
    ```
    NGROK_ADDRESS=https://b5d38d72752d.ngrok.io
    ```
- log in    
    ```
    PASSWORD=$(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode) && echo -n $PASSWORD | faas-cli login -g $NGROK_ADDRESS -u admin --password-stdin
    ```

## test remote URL
- list
    ```
    faas-cli list --gateway $NGROK_ADDRESS
    ```
    ```
    Function                      	Invocations    	Replicas
    astronaut-finder              	0              	1    
    base64                        	0              	1    
    echoit                        	0              	1    
    hello-openfaas                	0              	1    
    hubstats                      	0              	1    
    ingest-file                   	3              	1    
    logging-demo                  	4              	1    
    markdown                      	0              	1    
    nodeinfo                      	0              	1    
    sentimentanalysis             	2              	1    
    sorter                        	0              	1    
    wordcount                     	0              	1    
    workflow-demo                 	6              	1   
    ```

## create a webhook receiver **issue-bot**
- create function
    ```
    faas-cli new --lang python3 issue-bot --prefix=$OPENFAAS_PREFIX
    ```
- update issue-bot.yml
    ```
    provider:
        name: openfaas
        gateway: http://127.0.0.1:8080

    functions:
    issue-bot:
        lang: python3
        handler: ./issue-bot
        image: <user-name>/issue-bot
        environment:
        write_debug: true
    ```
- deploy
    ```
    sudo faas-cli build -f issue-bot.yml
    sudo faas-cli push -f issue-bot.yml
    sudo -E faas-cli deploy -f issue-bot.yml
    ```

## receive webhooks from Github
- create repo 'bot-tester'
- *Setting* -> *Webhooks* -> *Add Webhook*
    - Payload URL : https://b5d38d72752d.ngrok.io/function/issue-bot
    - Content Type: application/json
    - Leave Secret blank
    - Choose 'Let me select individual events'
        - select *Issues* and *Issue comment*

- create issue
- check the function is invoked
    ```
    faas-cli list
    ```
    ```
    Function                      	Invocations    	Replicas
    ...
    issue-bot                     	3              	1    
    ...
    ```
- check payload
    ```
    kubectl logs deployment/issue-bot -n openfaas-fn
    ```
    ```
    2020/07/08 15:53:18 Forking fprocess.
    2020/07/08 15:53:18 Query  
    2020/07/08 15:53:18 Path  /
    {"action":"opened","issue":{"url":"https://api.github.com/repos/sijoonlee/bot-tester/issues/3","repository_url":"https://api.github.com/repos/sijoonlee/bot-tester","labels_url":"https://api.github.com/repos/sijoonlee/bot-tester/issues/3/labels{/name}","comments_url":"https://api.github.com/repos/sijoonlee/bot-tester/issues/3/comments","events_url":"https://api.github.com/repos/sijoonlee/bot-tester/issues/3/events","html_url":"https://github.com/sijoonlee/bot-tester/issues/3","id":653409276,"node_id":"MDU6SXNzdWU2NTM0MDkyNzY=","number":3,"title":"test issue","user":{"login":"sijoonlee","id":41501772,"node_id":"MDQ6VXNlcjQxNTAxNzcy","avatar_url":"https://avatars3.githubusercontent.com/u/41501772?v=4","gravatar_id":"","url":"https://api.github.com/users/sijoonlee","html_url":"https://github.com/sijoonlee","followers_url":"https://api.github.com/users/sijoonlee/followers","following_url":"https://api.github.com/users/sijoonlee/following{/other_user}","gists_url":"https://api.github.com/users/sijoonlee/gists{/gist_id}","starred_url":"https://api.github.com/users/sijoonlee/starred{/owner}{/repo}","subscriptions_url":"https://api.github.com/users/sijoonlee/subscriptions","organizations_url":"https://api.github.com/users/sijoonlee/orgs","repos_url":"https://api.github.com/users/sijoonlee/repos","events_url":"https://api.github.com/users/sijoonlee/events{/privacy}","received_events_url":"https://api.github.com/users/sijoonlee/received_events","type":"User","site_admin":false},"labels":[],"state":"open","locked":false,"assignee":null,"assignees":[],"milestone":null,"comments":0,"created_at":"2020-07-08T15:53:17Z","updated_at":"2020-07-08T15:53:17Z","closed_at":null,"author_association":"OWNER","active_lock_reason":null,"body":"this is test"},
    ...
    ...
    2020/07/08 15:53:18 Duration: 0.032593 seconds
    ```

## Deploy SentimentAnalysis function
- get SentimentAnalysis function from store
    ```
    faas-cli store deploy SentimentAnalysis
    ```

## Update Issue-bot function
- edit handler.py
    ```
    import requests, json, os, sys

    def handle(req):

        event_header = os.getenv("Http_X_Github_Event")

        if not event_header == "issues":
            sys.exit("Unable to handle X-GitHub-Event: " + event_header)
            return

        gateway_hostname = os.getenv("gateway_hostname", "gateway")

        payload = json.loads(req)

        if not payload["action"] == "opened":
            return

        #sentimentanalysis
        res = requests.post('http://' + gateway_hostname + ':8080/function/sentimentanalysis', data=payload["issue"]["title"]+" "+payload["issue"]["body"])

        if res.status_code != 200:
            sys.exit("Error with sentimentanalysis, expected: %d, got: %d\n" % (200, res.status_code))

        return res.json()
    ```
- update requirements.txt
    ```
    requests
    ```
- add enviroment to issue-bot.yml
    ```
    gateway_hostname: "gateway.openfaas"
    ```
- deploy
    ```
    sudo faas-cli build -f issue-bot.yml
    sudo faas-cli push -f issue-bot.yml
    sudo -E faas-cli deploy -f issue-bot.yml
    ```  

## Check Response at Github
- *Setting* -> *Webhook* -> click the webhook -> scroll down -> *Recent Delivery* -> click the most recent one -> click *Response*  

## Create a Personal Access Token in Github
- GitHub profile -> Settings/Developer settings -> Personal access tokens and then click Generate new token
    - Note: openfaas-workshop
    - Select scopes: check repo
- Copy the token
- Create a file called env.yml in the directory where your issue-bot.yml file is located with the following content
    ```
    environment:
    auth_token: <auth_token_value>
    ```
- add env.yml to gitignore
- Update issue-bot.yml
    ```
    environment:
      write_debug: true
      gateway_hostname: "gateway.openfaas"
      positive_threshold: 0.25
    environment_file:
    - env.yml
    ```

## Update issue-bot function
- update requirements.txt
    ```
    PyGithub
    ```
- update handler
    ```
    import requests, json, os, sys
    from github import Github

    def handle(req):
        event_header = os.getenv("Http_X_Github_Event")

        if not event_header == "issues":
            sys.exit("Unable to handle X-GitHub-Event: " + event_header)
            return

        gateway_hostname = os.getenv("gateway_hostname", "gateway")

        payload = json.loads(req)

        if not payload["action"] == "opened":
            sys.exit("Action not supported: " + payload["action"])
            return

        # Call sentimentanalysis
        res = requests.post('http://' + gateway_hostname + ':8080/function/sentimentanalysis', 
                            data= payload["issue"]["title"]+" "+payload["issue"]["body"])

        if res.status_code != 200:
            sys.exit("Error with sentimentanalysis, expected: %d, got: %d\n" % (200, res.status_code))

        # Read the positive_threshold from configuration
        positive_threshold = float(os.getenv("positive_threshold", "0.2"))

        polarity = res.json()['polarity']

        # Call back to GitHub to apply a label
        apply_label(polarity,
            payload["issue"]["number"],
            payload["repository"]["full_name"],
            positive_threshold)

        return "Repo: %s, issue: %s, polarity: %f" % (payload["repository"]["full_name"], payload["issue"]["number"], polarity)

    def apply_label(polarity, issue_number, repo, positive_threshold):
        g = Github(os.getenv("auth_token"))
        repo = g.get_repo(repo)
        issue = repo.get_issue(issue_number)

        has_label_positive = False
        has_label_review = False
        for label in issue.labels:
            if label == "positive":
                has_label_positive = True
            if label == "review":
                has_label_review = True

        if polarity > positive_threshold and not has_label_positive:
            issue.set_labels("positive")
        elif not has_label_review:
            issue.set_labels("review")
    ```

- deploy
    ```
    sudo faas-cli build -f issue-bot.yml
    sudo faas-cli push -f issue-bot.yml
    sudo -E faas-cli deploy -f issue-bot.yml
    ```  