# Setting up a Podman Sites Demo

## Introduction

Podman Sites enable you to deploy an RHSI Router to a RHEL machine. This is different to a Gateway because it is a fully-fledged RHSI router and not just an end point.

Podman Sites are currently in Tech Preview and the features and doco are sketchy. For example, the console is completely non-functional with Podman Sites.

## Demo Overview

This demo will deploy NGinx at one site and make it accessible from another site. To demonstrate the power of this you should deploy NGinx at a location that does not normally have a routable network connection.

For the demo I will name the sites as ``North`` and ``South``. ``South`` can route to ``North``, but ``North`` cannot route to ``South``.

### Demo Setup
1. Ensure that no VMs have a firewall enabled.
2. Validate there is a routable network path from ```South``` to ```North```. (Try using PING.)
3. Deploy RHSI to the VMs in ``North`` and ``South.`` Instructions can be found at [Skupper.io](https://skupper.io./releases/index.html).
4. Deploy NGINX at the ```South``` site
    ```
    podman run --name mynginx -d -p 8000:80 nginx:latest
    ```
5. Check NGINX is running:
    ```
    curl localhost:8000
    ```
    Output should be:
    ```
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    ```

## Establishing the RHSI Network between the sites.

### North Site
Deploy the Podman Router
```
$ skupper init --site-name NORTH --platform podman --ingress-host <ip adress of the host>
```

Create a secret token that you will import at the South site:
```
$ skupper token create --platform podman north.token
```

Display the contents of the token and copy it to your clipboard.

```
$ cat north.token
```
Your token will look something like this (do not use this token in your example or the link creation will fail).
```
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURMVENDQWhXZ0F3SUJBZ0lSQU95NEEya2dmeDBFV1VmMklWeEQ2VEF3RFFZSktvWklodmNOQVFFTEJRQXcKR2pFWU1CWUdBMVVFQXhNUGMydDFjS
  <snip>
  NFUlRJRklDQVRFLS0tLS0K
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURTVENDQWpHZ0F3SUJBZ0lRTjVCeEZrbkhwYVliaUR1SHVyMWpNekFOQmdrcWhraUc5dzBCQVFzRkFEQWEKTVJnd0ZnWURWUVFERXc5emEzVndjR
  <snip>
  lRJRklDQVRFLS0tLS0K
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcFFJQkFBS0NBUUVBc2ZFUDBxVHM4cDhKRDRHczJaQlF1T2VKWnI3dStQT29BQk1hQkNjSVdIQjI3YUVqCkJkaHFhZzVkT2wvL1B6QzV5V
  <snip>  N6QnVHNy9WMG83U1JTWXJTUlg2Y1JMT1J0MHJOSTRQVzdnYVZqOUQ0PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
kind: Secret
metadata:
  annotations:
    edge-host: 10.10.20.42
    edge-port: "45671"
    inter-router-host: 10.10.20.42
    inter-router-port: "55671"
    skupper.io/generated-by: 1662c834-99ca-46eb-9e91-65431d01ffb6
    skupper.io/site-version: 1.4.3
  creationTimestamp: null
  labels:
    skupper.io/type: connection-token
  name: skupper
type: kubernetes.io/tls

```

### South Site

```skupper init --site-name NORTH --platform podman --ingress-host <ip adress of the host>```

Using your editor of choice, create a new file ```north.token```, paste the token contents from the ```North``` site and save the file.

Import the token

```
skupper link create north.token
```

Test the link is created
```
skupper status
```

At this ppiont you have two sites and the Service Interconnect network is established between them.

## Create the Service

### South Site
Create the RHSI service
```
skupper expose host host.containers.internal --port 8000 --address nginx --target-port 8000
```

Check the service is created
```
skupper service status
```

### North site

Expose the remote service locally:
```
skupper service create nginx 8000 --host-port 8000
```

Check you can curl the remote service:
```
curl localhost:8000
```