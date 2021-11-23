# Part2 Exercises

## 2.01:

**logoutput index.js**:
```
const express = require('express')
const app = express()
const port = 4000
const path = require('path')
const randomstring = require('randomstring');
const timestamp = require('time-stamp');
const fs = require('fs');
const axios = require('axios');

const directory = path.join('/', 'usr', 'src', 'app','log')

function getTimeStamp() {
  return timestamp.utc('[YYYY:MM:DD:mm:ss]') + " : " + randomstring.generate()
}

app.get('/', (req, res) => {
  let ts = getTimeStamp();
  let pings = '';
  let result = ts + " :: " + "Ping / Pongs: ";
  axios.get('http://pingpong-svc/pingpong')
  .then(function (response) {
    result += response.data.toString();
    res.send(result);
    // handle success
    console.log(response.data.toString());
  })
  .catch(function (error) {
    result += error;
    res.send(result);
    // handle error
    console.log(error);
  })
});

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
  setInterval(() => {
    var ts = getTimeStamp();
    console.log(ts)
  }, 5000);
})



```
**logoutput deployment.yaml:**
```
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
          image: mcprn/logoutput:0.12
```
**logoutput service.yaml:**
```
apiVersion: v1
kind: Service
metadata:
  name: logoutput-svc
spec:
  type: ClusterIP
  selector:
    app: logoutput
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 4000
```
**logoutput ingress.yaml:**
```
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
              number: 80
```
**ping-pong index.js:**
```
const express = require('express')
const app = express()
const port = 3300
let pings = 0;

app.get('/pingpong', (req, res) => {
  pings++;
  res.status(200).send(pings.toString())
})

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
})

```
**ping-pong deployment.yaml:**
```
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
      containers:
        - name: pingpong
          image: mcprn/ping-pong:0.12



```
**ping-pong service.yaml:**
```
apiVersion: v1
kind: Service
metadata:
  name: pingpong-svc
spec:
  type: ClusterIP
  selector:
    app: pingpong
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 3300

```
## 2.02:
**project ingress.yaml:**
```
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
      - path: /todos
        pathType: Prefix
        backend:
          service:
            name: database-svc
            port:
              number: 80
```
**project index.js:**
```
const express = require('express')
const app = express()
const port = 3000
const fs = require('fs');
const path = require('path')
const request = require('request');
const axios = require('axios');

const imageUrl = 'https://picsum.photos/600';
const directory = path.join('/', 'usr', 'src', 'app','public','images')
const imagePath = path.join(directory, 'image.jpg')

const backend = 'http://database-svc/todos';

let download = function(uri, filename, callback){
  request.head(uri, function(err, res, body){    
    request(uri).pipe(fs.createWriteStream(filename)).on('close', callback);
  });
};
var bodyParser = require('body-parser')
var cors = require('cors')
app.use(bodyParser.json())
app.use(cors())
app.use(express.static(path.join(__dirname, 'public')));


let showPage = function(req, res) {
  axios.get(backend)
  .then(function (response) {
    let todos = response.data;
    let todoHTML = '';
    for(let i = 0; i < todos.length; i++) {
      todoHTML += `<li>${todos[i].toString()}</li>`;
    }
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
          <div>
            <input type="text" id="todo" placeholder="Todo (max 140 chars)" name="todo" maxlength="140">
            <button onclick="send();">Add Todo</button>
          </div>
        <p><b>ToDos:</b></p>
        <ul>
          ${todoHTML}
        <ul>
      </div>
      <script>
        function send() {
          let todoEl = document.getElementById("todo");
          let todo = todoEl.value;
          let data = {todo};
          fetch("/", {
            method: "POST",
            headers: {'Content-Type': 'application/json'}, 
            body: JSON.stringify(data)
          }).then(res => {
            if(res.status == 200) {
              window.location.replace("/")
              //console.log(res)
            }
          })
        }
      </script>
    </body>
    </html>`;
    res.send(html);
  })
  .catch(function (error) {;
    // handle error
    console.log(error);
  })
};

app.post('/',  (req, res) => {
  console.log(req.body)
  const data = req.body;
  const headers = { 
      'Content-Type': 'application/json'
  };
  axios.post(backend, data, { headers })
  .then(response => {
      if(response.status == 200) {
        res.sendStatus(200)
      }
    });
});

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
```
**database index.js:**

```
const StormDB = require("stormdb");
const express = require('express')
var bodyParser = require('body-parser')
const app = express()
const port = 5200
var cors = require('cors')

app.use(bodyParser.json())
app.use(cors())
const engine = new StormDB.localFileEngine("./db.stormdb");
const db = new StormDB(engine);


db.default({ todos: [] });


app.post('/todos', cors(), (req, res) => {
  // console.log(req.body)
  const todo = req.body.todo;
  // console.log(todo)
  db.get("todos").push(todo);
  db.save();
  res.sendStatus(200);
});

app.get('/todos', (req, res) => {
  var todos = db.get("todos").state.todos;
  html = "<h2>Todos:</h2>";
  for(let i = 0; i < todos.length; i++) {
    html += `<p>${todos[i]}</p>`;
  }
  //console.log(todos);
  res.send(todos)
})

app.listen(port, () => {
  console.log(`Server started in port ${port}`)
})
```
## 2.03
**namespace command:**
```
kubectl create namespace logger
```
**all the yaml files:**
```
...
metadata:
  namespace: logger
...
```

## 2.04
**namespace command:**
```
kubectl create namespace webapp
```
**all the yaml files:**
```
...
metadata:
  namespace: webapp
...
```
## 2.05

Nothing to submit as per instructions. But I'll add my encrypted API key:
```
apiVersion: v1
kind: Secret
metadata:
    name: pixabay-apikey
data:
    API_KEY: ENC[AES256_GCM,data:3oq19jhFXll0sUbOR1xowK2gTZW8vRCRKgfEpoHVG5rgwsgKRgzzmje7Q4V6mVET,iv:mGR6EECpNC/m/cdVKcUacHABbflqbPJ7sDxROxZWV24=,tag:BQinKxBuaY7yta4Cm/ZZug==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age1ud8zms4fd008924sf90uv0wjyem8cgcqf6llgs07c6zn53cq3e7sk6wkgk
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBVN3h5NmF5WUdXK00yV3hh
            VVRhMlpHYkkrd2RrRDdFNlBKUVJ6MUQ5TkJBCkgrd2RLeVpKcytWYUpOWDErMThz
            TjR1MzVxdWpRR3FodXk5dmNtdE14b1kKLS0tIC94WUpzcWRYU05JOWprcVpGb2tU
            Y2xKc1pRSko0bnlGMVJCek93K2UxL1EK65cGwa2Tz7l2Mg1IrwyLFtuEdPMd5Svv
            q/8ZmG33REAy/87Ek46SB7eMoiCRrPhi3MQVaofJ/7Zt1e0B0mDJ0w==
            -----END AGE ENCRYPTED FILE-----
    lastmodified: "2021-11-23T19:58:57Z"
    mac: ENC[AES256_GCM,data:SMHBszRgOJnWtLWNWLD5nsFBZ0uj+I3A3h8D0/aYOCqaHX+zplyPu1noPTNUq9rhY2M/HSolqqIibYMUw6FxAloyxzpp4xBfvUr+B+RIzEFl+S7ai3gO/LhOY85YuiK9oYl92CQw66Vidit53OQiAfqM9LAmJjnJ85G7YwWtcX8=,iv:xHwmsML5bnTeoxJury7FZMlfqnwdtBqzNcLo5GdABHg=,tag:eND8lIIMjZJIO+b8zo7zgw==,type:str]
    pgp: []
    encrypted_regex: ^(data)$
    version: 3.7.1
```
## 2.06

logoutput **index.js**:
```
...
const MESSAGE = process.env.MESSAGE || "";
...
console.log(MESSAGE);
...
```

in **deployment.yaml**:
```
...
spec:
      containers:
        - name: logoutput
          image: mcprn/logoutput:0.13
          env:
          - name: MESSAGE
            value: "Hello there."
...
```

**command**:
```
kubectl logs -f logoutput-dep-5bf84b66f-6zgk7 --namespace=logger
```
**outputs**:
```
Hello there.
Server started in port 4000
[2021:11:23:25:21] : Xi7oazXPS6pOCCpL41hPa2shSbx3jxXc
[2021:11:23:25:26] : uwqEzBGb4VBf6IrTqjRnfJbn5oMGlZ45
....
```
## 2.07

