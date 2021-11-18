Exercises 

1.01:

Dockerfile:

FROM alpine:3.13
WORKDIR /usr/src/app
COPY script.sh /usr/src/app/script.sh
RUN chmod +x /usr/src/app/script.sh && apk add util-linux moreutils 
CMD sh /usr/src/app/script.sh

script.sh:
#!/bin/bash
uuid=$(uuidgen)
echo $uuid | ts '[%Y-%m-%d %H:%M:%.S]' 
while sleep 5; do echo $uuid | ts '[%Y-%m-%d %H:%M:%.S]'; done

commands:
docker build . -t logoutput
docker tag logoutput mcprn/logoutput
docker push mcprn/logoutput:latest

kubectl create deployment logoutput-dep --image=mcprn/logoutput 

kubectl logs -f logoutput-dep-79c84dfbf4-d85mn 


outputs:
...
[2021-11-16 20:30:04.314763] e24bd8af-41d2-4948-85d5-aa88d2a0780e
[2021-11-16 20:30:09.379689] e24bd8af-41d2-4948-85d5-aa88d2a0780e
[2021-11-16 20:30:14.448124] e24bd8af-41d2-4948-85d5-aa88d2a0780e
...

1.02:
Dockerfile:

FROM alpine:3.13
WORKDIR /usr/src/app
COPY index.js .
COPY package.json .
RUN apk add --update nodejs && apk add --update npm && npm install 
CMD node index.js

index.js:

const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => {
  res.send('')
})

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
})

package.json:

{
  "name": "project",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.17.1"
  }
}

commands:
docker build . -t project
docker tag project mcprn/project
docker push mcprn/project:latest
kubectl create deployment project-dep --image=mcprn/project


1.03:

deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: logoutput-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logoutput
  template:
    metadata:
      labels:
        app: logoutput
    spec:
      containers:
        - name: logoutput
          image: mcprn/logoutput:latest

1.04:

deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: project
  template:
    metadata:
      labels:
        app: project
    spec:
      containers:
        - name: project
          image: mcprn/project:latest

1.05:

I updated the index.js to return simple HTML page, and updated the image to registry with a new tag. Then I modified the reference of the image in the deployment.yaml to correspond the new tag.
Next I applied the new configuration and forwarded the port with:
kubectl port-forward hashresponse-dep-57bcc888d7-dj5vk 3003:3000

Then my page was visible in the localhost:3003

1.06:

service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: project-svc
spec:
  type: NodePort
  selector:
    app: project
  ports:
    - name: http
      nodePort: 30080
      protocol: TCP
      port: 3003
      targetPort: 3000

The file service.yaml is place in manifests directory with old deployment.yaml. Commands ran:
k3d cluster create --port 8082:30080@agent:0 -p 8081:80@loadbalancer --agents 2
kubectl apply -f manifests/service.yaml 
kubectl apply -f manifests/deployment.yaml

Now the application is available in localhost:8082

1.07:

For this one I had to make modifications to my log output app. I changed it nodejs / express app.

index.js

const express = require('express')
const app = express()
const port = 4000
const randomstring = require('randomstring');
const timestamp = require('time-stamp');

function getTimeStamp() {
  return timestamp.utc('[YYYY:MM:DD:mm:ss]') + " : " + randomstring.generate()
}

app.get('/', (req, res) => {
  res.send(getTimeStamp())
})

console.log()

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
  setInterval(() => {console.log(getTimeStamp())}, 5000);
})



package.json:

{
  "name": "logoutput",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.17.1",
    "randomstring": "^1.2.1",
    "time-stamp": "^2.2.0"
  }
}

deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: logoutput-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logoutput
  template:
    metadata:
      labels:
        app: logoutput
    spec:
      containers:
        - name: logoutput
          image: mcprn/logoutput:0.3

ingress.yaml:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcprn-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: logoutput-svc
            port:
              number: 2345

service.yaml

apiVersion: v1
kind: Service
metadata:
  name: logoutput-svc
spec:
  type: ClusterIP
  selector:
    app: logoutput
  ports:
    - port: 2345
      protocol: TCP
      targetPort: 4000

updated the image:

docker build . -t logoutput:0.3
docker tag logoutput:0.3 mcprn/logoutput:0.3
docker push mcprn/logoutput:0.3

and applied the change (all the yaml files are in manifests folder):
kubectl apply -f manifests/

This way I could access to localhost:8081 and see the timestamp and random string in browser.

1.08:

service.yaml

apiVersion: v1
kind: Service
metadata:
  name: project-svc
spec:
  type: ClusterIP
  selector:
    app: project
  ports:
    - port: 2345
      protocol: TCP
      targetPort: 3000

ingress.yaml:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcprn-ingress-project
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: project-svc
            port:
              number: 2345

The deployment.yaml remained unchanged. I removed the logoutput configurations and applied these. Then localhost:8081 responded with a web page.

1.09:

I created a new application ping-pong:

package.json:

{
  "name": "ping-pong",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.17.1"
  }
}

index.js:

const express = require('express')
const app = express()
const port = 3300
let reqs = 0;
app.all('*', (req, res) => {
  res.send('<p>pong ' + reqs + ' </p>')
  reqs ++;
})

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
})

Built and pushed the image:

docker build . -t ping-pong
docker tag ping-pong mcprn/ping-pong
docker push mcprn/ping-pong:latest

then I stopped all services and ingresses.

I had one ingress.yaml:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcprn-ingresss
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: logoutput-svc
            port:
              number: 2345
      - path: /pingpong
        pathType: Prefix
        backend:
          service:
            name: pingpong-svc
            port:
              number: 2346

pingpong service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: pingpong-svc
spec:
  type: ClusterIP
  selector:
    app: pingpong
  ports:
    - port: 2346
      protocol: TCP
      targetPort: 3300

logoutput service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: logoutput-svc
spec:
  type: ClusterIP
  selector:
    app: logoutput
  ports:
    - port: 2345
      protocol: TCP
      targetPort: 4000

Now when all started, I can access.
localhost:8081 = log output
localhost:8081/pingpong = pong + current count

1.10:

deployment.yaml:


apiVersion: apps/v1
kind: Deployment
metadata:
  name: logread-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logread
  template:
    metadata:
      labels:
        app: logread
    spec:
      volumes: # Define volume
        - name: shared-image
          emptyDir: {}
      containers:
        - name: logread-container
          image: mcprn/logread:0.6
          volumeMounts: # Mount volume
          - name: shared-image
            mountPath: /usr/src/app/log
        - name: logoutput-container
          image: mcprn/logoutput:0.6
          volumeMounts: # Mount volume
          - name: shared-image
            mountPath: /usr/src/app/log

logoutput index.js:

const express = require('express')
const app = express()
const port = 4000
const path = require('path')
const randomstring = require('randomstring');
const timestamp = require('time-stamp');
const fs = require('fs');

const directory = path.join('/', 'usr', 'src', 'app','log')
const filePath = path.join(directory, 'server.log')

function getTimeStamp() {
  return timestamp.utc('[YYYY:MM:DD:mm:ss]') + " : " + randomstring.generate()
}

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
  setInterval(() => {
    var ts = getTimeStamp();
    fs.writeFileSync(filePath, ts);
    console.log(ts)
  }, 5000);
})

logread index.js:

const express = require('express')
const app = express()
const port = 4400
const randomstring = require('randomstring');
const timestamp = require('time-stamp');
const fs = require('fs');
const path = require('path')

const directory = path.join('/', 'usr', 'src', 'app','log')
const filePath = path.join(directory, 'server.log')

app.get('/', (req, res) => {
  const fs = require('fs')
  let data = '';
  try {
    data = fs.readFileSync(filePath, 'utf8')
  } catch (err) {
    data = 'Error reading data';
    console.error(err)

  }
  res.send(data)
})


app.listen(port, () => {
  console.log(`Server started in port ${port}`)
})


1.11:

ping-pong index.js:

const express = require('express')
const app = express()
const port = 3300
let pings = 0;
const fs = require('fs');
const path = require('path')

const directory = path.join('/', 'usr', 'src', 'app','log')
const pingPath = path.join(directory, 'ping.log')

app.get('/pingpong', (req, res) => {
  pings++;
  fs.writeFile(pingPath, pings.toString(), (err) => { if (err) throw err; });
  res.send('<p>pong ' + pings + ' </p>');
})

app.listen(port, () => {
  pings = 0;
  try {
    pings = Number(fs.readFileSync(pingPath, 'utf8'))
  } catch (err) {
    pings = 0;
    console.error(err)
  }
  console.log(`Server started in port ${port}`)
})

pingpong deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: pingpong-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pingpong
  template:
    metadata:
      labels:
        app: pingpong
    spec:
      volumes:
        - name: shared-vol
          persistentVolumeClaim:
            claimName: vol-claim
      containers:
        - name: pingpong
          image: mcprn/ping-pong:0.10
          volumeMounts:
          - name: shared-vol
            mountPath: /usr/src/app/log
        - name: logoutput
          image: mcprn/logoutput:0.7
          volumeMounts:
          - name: shared-vol
            mountPath: /usr/src/app/log


pingpong persistentvolume.yaml:

apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  storageClassName: manual
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  local:
    path: /tmp/kube
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k3d-k3s-default-agent-0

pingpong persistentvolumeclaim.yaml:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vol-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi

logoutput index.js:

const express = require('express')
const app = express()
const port = 4000
const path = require('path')
const randomstring = require('randomstring');
const timestamp = require('time-stamp');
const fs = require('fs');

const directory = path.join('/', 'usr', 'src', 'app','log')
const logPath = path.join(directory, 'server.log')
const pingPath = path.join(directory, 'ping.log')

function getTimeStamp() {
  return timestamp.utc('[YYYY:MM:DD:mm:ss]') + " : " + randomstring.generate()
}

app.get('/', (req, res) => {
  let ts = '';
  let pings = '';
  try {
    ts = fs.readFileSync(logPath, 'utf8')
  } catch (err) {
    ts = 'Error reading timestamp data';
    console.error(err)
  }
  try {
    pings = fs.readFileSync(pingPath, 'utf8')
  } catch (err) {
    pings = 'Error reading ping data';
    console.error(err)
  }
  let result = ts + " :: " + "Ping / Pongs: " + pings;
  res.send(result);
});

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
  setInterval(() => {
    var ts = getTimeStamp();
    fs.writeFileSync(logPath, ts);
    console.log(ts)
  }, 5000);
})


1.12:

project index.js:

const express = require('express')
const app = express()
const port = 3000
const fs = require('fs');
const path = require('path')
const request = require('request');

const imageUrl = 'https://picsum.photos/600';
const directory = path.join('/', 'usr', 'src', 'app','public','images')
const imagePath = path.join(directory, 'image.jpg')


let download = function(uri, filename, callback){
  request.head(uri, function(err, res, body){    
    request(uri).pipe(fs.createWriteStream(filename)).on('close', callback);
  });
};

app.use(express.static(path.join(__dirname, 'public')));


let showPage = function(req, res) {
  let html = `<!doctype html>
  <html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Project</title>
    <meta name="description" content="The Project">
  </head>
  
  <body>
    <div style="text-align: center;">
      <img src="/images/image.jpg">
    </div>
  </body>
  </html>`;
  res.send(html);
  
};


app.get('/', (req, res) => {
  if (fs.existsSync(imagePath)) {
    console.log("yes")
    showPage(req, res);
  } else {
    download(imageUrl, imagePath, function(){
      console.log('doewnload done');
      showPage(req, res);
    });
    console.log("no image, download...")
  }
})

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
})


deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: project
  template:
    metadata:
      labels:
        app: project
    spec:
      volumes:
        - name: shared-vol
          persistentVolumeClaim:
            claimName: vol-claim
      containers:
        - name: pingpong
          image: mcprn/project:0.8
          volumeMounts:
          - name: shared-vol
            mountPath: /usr/src/app/public/images

1.13:


project index.js:

const express = require('express')
const app = express()
const port = 3000
const fs = require('fs');
const path = require('path')
const request = require('request');

const imageUrl = 'https://picsum.photos/600';
const directory = path.join('/', 'usr', 'src', 'app','public','images')
const imagePath = path.join(directory, 'image.jpg')


let download = function(uri, filename, callback){
  request.head(uri, function(err, res, body){    
    request(uri).pipe(fs.createWriteStream(filename)).on('close', callback);
  });
};

app.use(express.static(path.join(__dirname, 'public')));


let showPage = function(req, res) {
  let html = `<!doctype html>
  <html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Project</title>
    <meta name="description" content="The Project">
  </head>
  
  <body>
    <div style="text-align: center;">
      <img style="max-height: 400px;" src="/images/image.jpg">
      <form>
        <input type="text" id="todo" placeholder="Todo (max 140 chars)" name="todo" maxlength="140">
        <button>Add Todo</button>
      </form>
      <p><b>ToDos:</b></p>
      <ul>
        <li>Be nice to Mom</li>
        <li>Pet your cat</li>
        <li>Eat a healthy lunch</li>
      <ul>
    </div>
  </body>
  </html>`;
  res.send(html);
  
};

app.get('/', (req, res) => {
  if (fs.existsSync(imagePath)) {
    console.log("Image exists")
    showPage(req, res);
  } else {
    download(imageUrl, imagePath, function(){
      console.log('Download done');
      showPage(req, res);
    });
    console.log("No image, download...")
  }
})

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
})


















