# HTTP和HTTPS服务器

## HTTP 模块概述

Node.js 的 `http` 和 `https` 模块提供了创建 HTTP/HTTPS 服务器和客户端的功能。这些模块是构建 Web 应用的基础。

### 与其他语言的对比

| 特性 | Node.js http | Python Flask/FastAPI | Java Spring Boot | Go net/http |
|------|--------------|---------------------|------------------|-------------|
| 服务器类型 | 原生模块 | 框架 | 框架 | 标准库 |
| 异步模型 | 事件驱动 | async/await | 多线程 | goroutine |
| 中间件 | 手动实现 | 支持 | 支持 | 手动实现 |
| 路由 | 手动实现 | 装饰器 | 注解 | 手动实现 |

## 创建 HTTP 服务器

### 基本 HTTP 服务器

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // 设置响应头
  res.setHeader('Content-Type', 'text/plain; charset=utf-8');
  res.statusCode = 200;
  
  // 发送响应
  res.end('Hello, Node.js!\n');
});

server.listen(3000, 'localhost', () => {
  console.log('服务器运行在 http://localhost:3000');
});

// 优雅关闭
process.on('SIGTERM', () => {
  server.close(() => {
    console.log('服务器已关闭');
  });
});
```

### 处理不同路由

```javascript
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
  const parsedUrl = url.parse(req.url, true);
  const path = parsedUrl.pathname;
  const method = req.method;
  
  // 设置默认响应头
  res.setHeader('Content-Type', 'application/json; charset=utf-8');
  
  // 路由处理
  if (path === '/api/users' && method === 'GET') {
    res.statusCode = 200;
    res.end(JSON.stringify([
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ]));
  } else if (path === '/api/users' && method === 'POST') {
    let body = '';
    
    req.on('data', chunk => {
      body += chunk.toString();
    });
    
    req.on('end', () => {
      try {
        const user = JSON.parse(body);
        res.statusCode = 201;
        res.end(JSON.stringify({ message: '用户创建成功', user }));
      } catch (err) {
        res.statusCode = 400;
        res.end(JSON.stringify({ error: '无效的 JSON' }));
      }
    });
  } else if (path === '/api/users/:id' && method === 'GET') {
    const id = path.split('/').pop();
    res.statusCode = 200;
    res.end(JSON.stringify({ id, name: 'User ' + id }));
  } else {
    res.statusCode = 404;
    res.end(JSON.stringify({ error: '未找到' }));
  }
});

server.listen(3000, () => {
  console.log('服务器运行在 http://localhost:3000');
});
```

**与其他语言对比**：
- **Python Flask**: 使用装饰器 `@app.route('/api/users')`
- **Java Spring**: 使用 `@RequestMapping` 注解
- **Go**: 使用 `http.HandleFunc()` 注册路由

## 请求处理

### 读取请求数据

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // 获取请求方法
  console.log('请求方法:', req.method);
  
  // 获取请求 URL
  console.log('请求 URL:', req.url);
  
  // 获取请求头
  console.log('请求头:', req.headers);
  console.log('Content-Type:', req.headers['content-type']);
  
  // 读取请求体
  let body = '';
  
  req.on('data', chunk => {
    body += chunk.toString();
  });
  
  req.on('end', () => {
    console.log('请求体:', body);
    
    // 根据 Content-Type 解析
    const contentType = req.headers['content-type'] || '';
    
    if (contentType.includes('application/json')) {
      try {
        const data = JSON.parse(body);
        console.log('解析的 JSON:', data);
      } catch (err) {
        console.error('JSON 解析失败:', err);
      }
    } else if (contentType.includes('application/x-www-form-urlencoded')) {
      const params = new URLSearchParams(body);
      console.log('表单数据:', Object.fromEntries(params));
    }
    
    res.statusCode = 200;
    res.end('数据接收成功');
  });
  
  req.on('error', err => {
    console.error('请求错误:', err);
    res.statusCode = 400;
    res.end('请求错误');
  });
});

server.listen(3000);
```

### 流式处理请求

```javascript
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/upload') {
    const writeStream = fs.createWriteStream('uploaded-file.txt');
    
    // 直接管道传输，不加载到内存
    req.pipe(writeStream);
    
    req.on('end', () => {
      res.statusCode = 200;
      res.end('文件上传成功');
    });
    
    req.on('error', err => {
      writeStream.destroy();
      res.statusCode = 500;
      res.end('上传失败');
    });
  } else {
    res.statusCode = 404;
    res.end('未找到');
  }
});

server.listen(3000);
```

## 响应处理

### 设置响应头

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // 设置单个响应头
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.setHeader('X-Custom-Header', 'custom-value');
  
  // 设置多个响应头
  res.writeHead(200, {
    'Content-Type': 'application/json',
    'Cache-Control': 'no-cache',
    'Access-Control-Allow-Origin': '*'
  });
  
  res.end(JSON.stringify({ message: 'Hello' }));
});

server.listen(3000);
```

### 流式响应

```javascript
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  if (req.url === '/large-file') {
    // 流式传输大文件
    const fileStream = fs.createReadStream('large-file.txt');
    
    res.setHeader('Content-Type', 'text/plain');
    fileStream.pipe(res);
    
    fileStream.on('error', err => {
      if (!res.headersSent) {
        res.statusCode = 500;
        res.end('文件读取失败');
      }
    });
  } else {
    res.statusCode = 404;
    res.end('未找到');
  }
});

server.listen(3000);
```

### 分块传输

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/plain',
    'Transfer-Encoding': 'chunked'
  });
  
  // 分块发送数据
  res.write('第一块数据\n');
  
  setTimeout(() => {
    res.write('第二块数据\n');
  }, 1000);
  
  setTimeout(() => {
    res.write('第三块数据\n');
    res.end();  // 结束响应
  }, 2000);
});

server.listen(3000);
```

## HTTPS 服务器

### 创建 HTTPS 服务器

```javascript
const https = require('https');
const fs = require('fs');

// 读取 SSL 证书和密钥
const options = {
  key: fs.readFileSync('private-key.pem'),
  cert: fs.readFileSync('certificate.pem'),
  // 可选：CA 证书
  // ca: fs.readFileSync('ca-certificate.pem')
};

const server = https.createServer(options, (req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('HTTPS 服务器响应\n');
});

server.listen(443, () => {
  console.log('HTTPS 服务器运行在 https://localhost:443');
});
```

### 自签名证书生成

```bash
# 生成私钥
openssl genrsa -out private-key.pem 2048

# 生成证书签名请求
openssl req -new -key private-key.pem -out csr.pem

# 生成自签名证书
openssl x509 -req -days 365 -in csr.pem -signkey private-key.pem -out certificate.pem
```

**与其他语言对比**：
- **Python**: 使用 `ssl` 模块包装 socket
- **Java**: 使用 `SSLContext` 配置 HTTPS
- **Go**: 使用 `tls.Config` 配置 HTTPS

## HTTP 客户端

### 发送 GET 请求

```javascript
const http = require('http');

function httpGet(url) {
  return new Promise((resolve, reject) => {
    http.get(url, (res) => {
      let data = '';
      
      res.on('data', chunk => {
        data += chunk.toString();
      });
      
      res.on('end', () => {
        if (res.statusCode === 200) {
          resolve(data);
        } else {
          reject(new Error(`HTTP ${res.statusCode}`));
        }
      });
    }).on('error', reject);
  });
}

// 使用
(async () => {
  try {
    const response = await httpGet('http://api.example.com/data');
    console.log('响应:', response);
  } catch (err) {
    console.error('请求失败:', err);
  }
})();
```

### 发送 POST 请求

```javascript
const http = require('http');

function httpPost(url, data) {
  return new Promise((resolve, reject) => {
    const urlObj = new URL(url);
    
    const options = {
      hostname: urlObj.hostname,
      port: urlObj.port || 80,
      path: urlObj.pathname,
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Content-Length': Buffer.byteLength(data)
      }
    };
    
    const req = http.request(options, (res) => {
      let responseData = '';
      
      res.on('data', chunk => {
        responseData += chunk.toString();
      });
      
      res.on('end', () => {
        if (res.statusCode >= 200 && res.statusCode < 300) {
          resolve(responseData);
        } else {
          reject(new Error(`HTTP ${res.statusCode}: ${responseData}`));
        }
      });
    });
    
    req.on('error', reject);
    req.write(data);
    req.end();
  });
}

// 使用
(async () => {
  try {
    const response = await httpPost(
      'http://api.example.com/users',
      JSON.stringify({ name: 'Alice', email: 'alice@example.com' })
    );
    console.log('响应:', response);
  } catch (err) {
    console.error('请求失败:', err);
  }
})();
```

## 实际应用场景

### 场景 1：RESTful API 服务器

```javascript
const http = require('http');
const url = require('url');

class RESTfulServer {
  constructor() {
    this.routes = {
      GET: {},
      POST: {},
      PUT: {},
      DELETE: {}
    };
  }
  
  get(path, handler) {
    this.routes.GET[path] = handler;
  }
  
  post(path, handler) {
    this.routes.POST[path] = handler;
  }
  
  put(path, handler) {
    this.routes.PUT[path] = handler;
  }
  
  delete(path, handler) {
    this.routes.DELETE[path] = handler;
  }
  
  async handleRequest(req, res) {
    const parsedUrl = url.parse(req.url, true);
    const path = parsedUrl.pathname;
    const method = req.method;
    const routeHandlers = this.routes[method] || {};
    const handler = routeHandlers[path];
    
    res.setHeader('Content-Type', 'application/json');
    
    if (!handler) {
      res.statusCode = 404;
      res.end(JSON.stringify({ error: '未找到' }));
      return;
    }
    
    // 读取请求体
    let body = '';
    req.on('data', chunk => body += chunk.toString());
    
    req.on('end', async () => {
      try {
        const requestData = body ? JSON.parse(body) : {};
        const result = await handler({
          params: parsedUrl.query,
          body: requestData,
          headers: req.headers
        });
        
        res.statusCode = 200;
        res.end(JSON.stringify(result));
      } catch (err) {
        res.statusCode = 500;
        res.end(JSON.stringify({ error: err.message }));
      }
    });
  }
  
  listen(port) {
    const server = http.createServer((req, res) => {
      this.handleRequest(req, res);
    });
    
    server.listen(port, () => {
      console.log(`服务器运行在 http://localhost:${port}`);
    });
    
    return server;
  }
}

// 使用
const app = new RESTfulServer();

// 模拟数据存储
let users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
];

app.get('/api/users', () => users);

app.get('/api/users/:id', ({ params }) => {
  const id = parseInt(params.id);
  return users.find(u => u.id === id) || null;
});

app.post('/api/users', ({ body }) => {
  const newUser = { id: users.length + 1, ...body };
  users.push(newUser);
  return newUser;
});

app.listen(3000);
```

### 场景 2：文件上传服务器

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');
const { pipeline } = require('stream');

const server = http.createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/upload') {
    const fileName = req.headers['x-filename'] || 'uploaded-file';
    const filePath = path.join(__dirname, 'uploads', fileName);
    
    // 确保上传目录存在
    fs.mkdirSync(path.dirname(filePath), { recursive: true });
    
    const writeStream = fs.createWriteStream(filePath);
    
    pipeline(req, writeStream, (err) => {
      if (err) {
        res.statusCode = 500;
        res.end(JSON.stringify({ error: '上传失败' }));
      } else {
        res.statusCode = 200;
        res.end(JSON.stringify({ message: '上传成功', file: fileName }));
      }
    });
  } else if (req.method === 'GET' && req.url.startsWith('/download/')) {
    const fileName = req.url.split('/').pop();
    const filePath = path.join(__dirname, 'uploads', fileName);
    
    fs.access(filePath, fs.constants.F_OK, (err) => {
      if (err) {
        res.statusCode = 404;
        res.end('文件不存在');
      } else {
        const fileStream = fs.createReadStream(filePath);
        res.setHeader('Content-Disposition', `attachment; filename="${fileName}"`);
        fileStream.pipe(res);
      }
    });
  } else {
    res.statusCode = 404;
    res.end('未找到');
  }
});

server.listen(3000);
```

### 场景 3：代理服务器

```javascript
const http = require('http');
const https = require('https');
const url = require('url');

const proxyServer = http.createServer((req, res) => {
  const targetUrl = req.headers['x-target-url'] || 'https://api.example.com';
  const parsedUrl = url.parse(targetUrl);
  const protocol = parsedUrl.protocol === 'https:' ? https : http;
  
  const options = {
    hostname: parsedUrl.hostname,
    port: parsedUrl.port || (parsedUrl.protocol === 'https:' ? 443 : 80),
    path: parsedUrl.path,
    method: req.method,
    headers: {
      ...req.headers,
      host: parsedUrl.hostname
    }
  };
  
  const proxyReq = protocol.request(options, (proxyRes) => {
    res.writeHead(proxyRes.statusCode, proxyRes.headers);
    proxyRes.pipe(res);
  });
  
  proxyReq.on('error', (err) => {
    res.statusCode = 500;
    res.end('代理错误');
  });
  
  req.pipe(proxyReq);
});

proxyServer.listen(3000, () => {
  console.log('代理服务器运行在 http://localhost:3000');
});
```

### 场景 4：WebSocket 升级（基础）

```javascript
const http = require('http');
const crypto = require('crypto');

const server = http.createServer((req, res) => {
  if (req.url === '/ws' && req.headers.upgrade === 'websocket') {
    // WebSocket 握手
    const key = req.headers['sec-websocket-key'];
    const accept = crypto
      .createHash('sha1')
      .update(key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11')
      .digest('base64');
    
    res.writeHead(101, {
      'Upgrade': 'websocket',
      'Connection': 'Upgrade',
      'Sec-WebSocket-Accept': accept
    });
    
    // 注意：实际应用中应使用 ws 库
    res.end();
  } else {
    res.statusCode = 404;
    res.end('未找到');
  }
});

server.listen(3000);
```

## 最佳实践

### 1. 错误处理

```javascript
const server = http.createServer((req, res) => {
  try {
    // 处理请求
  } catch (err) {
    if (!res.headersSent) {
      res.statusCode = 500;
      res.setHeader('Content-Type', 'application/json');
      res.end(JSON.stringify({ error: '服务器错误' }));
    }
  }
});

server.on('error', (err) => {
  console.error('服务器错误:', err);
});
```

### 2. 超时设置

```javascript
const server = http.createServer((req, res) => {
  // 设置请求超时
  req.setTimeout(30000, () => {
    if (!res.headersSent) {
      res.statusCode = 408;
      res.end('请求超时');
    }
    req.destroy();
  });
  
  // 处理请求...
});

// 服务器超时
server.timeout = 120000;  // 2分钟
```

### 3. 连接限制

```javascript
const server = http.createServer((req, res) => {
  // 处理请求
});

server.maxHeadersCount = 2000;  // 最大请求头数量
server.headersTimeout = 40000;   // 请求头超时
```

### 4. 使用中间件模式

```javascript
class MiddlewareServer {
  constructor() {
    this.middlewares = [];
  }
  
  use(middleware) {
    this.middlewares.push(middleware);
  }
  
  async handleRequest(req, res) {
    let index = 0;
    
    const next = async () => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        await middleware(req, res, next);
      }
    };
    
    await next();
  }
}

// 使用
const app = new MiddlewareServer();

app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

app.use((req, res, next) => {
  res.setHeader('X-Powered-By', 'Node.js');
  next();
});

const server = http.createServer((req, res) => {
  app.handleRequest(req, res);
});
```

## 总结

- **http/https 模块**是构建 Web 服务器的基础
- 使用**流**处理大文件和请求体，避免内存溢出
- 正确设置**响应头**和**状态码**
- 实现**错误处理**和**超时控制**
- 使用**中间件模式**组织代码
- HTTPS 需要**SSL 证书**配置
- 考虑使用框架（如 Express）简化开发

