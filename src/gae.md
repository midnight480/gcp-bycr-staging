* app.yaml

```
runtime: nodejs10
```

* index.js
  
```
const http = require('http');
const httpProxy = require('http-proxy');
const port = process.env.PORT || 8080;
httpProxy.createProxyServer({target: 'http://internal.${domain}:8080/'}).listen(port);
```

* package.json

```
{
  "name": "gae-proxy",
  "version": "1.0.0",
  "description": "proxy",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "author": "Unknown",
  "license": "MIT",
  "dependencies": {
    "http": "0.0.1-security",
    "http-proxy": "^1.18.1"
  }
}
```

