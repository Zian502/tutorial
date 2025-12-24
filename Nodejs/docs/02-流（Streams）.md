# 流（Streams）

## 流的概念

流是 Node.js 中处理流式数据的抽象接口。流可以将数据分块处理，而不是一次性加载到内存中，这使得处理大文件或实时数据变得高效。

### 流的类型

Node.js 提供了四种类型的流：

1. **Readable**：可读流（如 `fs.createReadStream()`）
2. **Writable**：可写流（如 `fs.createWriteStream()`）
3. **Duplex**：双工流（可读可写，如 TCP socket）
4. **Transform**：转换流（双工流的特例，用于数据转换，如 `zlib.createGzip()`）

### 与其他语言的对比

| 特性 | Node.js Streams | Python | Java | Go |
|------|----------------|--------|------|-----|
| 流类型 | Readable/Writable/Duplex/Transform | io.BufferedReader/Writer | InputStream/OutputStream | io.Reader/Writer |
| 背压处理 | 自动处理 | 手动处理 | 手动处理 | Channel 缓冲 |
| 管道操作 | `pipe()` 方法 | `shutil.copyfileobj()` | `Files.copy()` | `io.Copy()` |
| 事件驱动 | 是 | 可选 | 可选 | 否 |

## 可读流（Readable Stream）

### 基本用法

```javascript
const fs = require('fs');

// 创建可读流
const readableStream = fs.createReadStream('large-file.txt', {
  encoding: 'utf8',
  highWaterMark: 64 * 1024  // 缓冲区大小：64KB
});

// 方式 1：监听 'data' 事件
readableStream.on('data', (chunk) => {
  console.log(`接收到 ${chunk.length} 字节数据`);
  // 处理数据块
});

readableStream.on('end', () => {
  console.log('数据读取完成');
});

readableStream.on('error', (err) => {
  console.error('读取错误:', err);
});

// 方式 2：使用 async iteration (Node.js 10+)
async function readStream() {
  try {
    for await (const chunk of readableStream) {
      console.log(`接收到 ${chunk.length} 字节数据`);
      // 处理数据块
    }
    console.log('数据读取完成');
  } catch (err) {
    console.error('读取错误:', err);
  }
}
```

### 暂停和恢复

```javascript
const readableStream = fs.createReadStream('file.txt');

readableStream.on('data', (chunk) => {
  console.log('接收到数据:', chunk);
  
  // 暂停读取
  readableStream.pause();
  
  // 1秒后恢复读取
  setTimeout(() => {
    readableStream.resume();
  }, 1000);
});
```

**与其他语言对比**：
- **Python**: `file.read(size)` 或 `file.readline()` 逐块读取
- **Java**: `BufferedReader.readLine()` 或 `read(byte[])` 方法
- **Go**: `io.Reader.Read()` 方法读取数据块

## 可写流（Writable Stream）

### 基本用法

```javascript
const fs = require('fs');

// 创建可写流
const writableStream = fs.createWriteStream('output.txt', {
  encoding: 'utf8',
  highWaterMark: 64 * 1024
});

// 写入数据
writableStream.write('第一行数据\n');
writableStream.write('第二行数据\n');
writableStream.write('第三行数据\n');

// 结束写入
writableStream.end('最后一行数据\n');

// 监听事件
writableStream.on('finish', () => {
  console.log('数据写入完成');
});

writableStream.on('error', (err) => {
  console.error('写入错误:', err);
});

// 检查是否可写
if (writableStream.writable) {
  writableStream.write('数据');
}
```

### 背压处理（Backpressure）

```javascript
const writableStream = fs.createWriteStream('output.txt');

function writeData(data) {
  if (!writableStream.write(data)) {
    // 缓冲区已满，暂停读取
    console.log('缓冲区已满，等待 drain 事件');
    writableStream.once('drain', () => {
      console.log('缓冲区已清空，可以继续写入');
      // 继续写入数据
    });
  }
}

// 使用示例
for (let i = 0; i < 1000000; i++) {
  writeData(`数据行 ${i}\n`);
}
```

**与其他语言对比**：
- **Python**: `file.write()` 可能阻塞，需要检查返回值
- **Java**: `OutputStream.write()` 可能阻塞
- **Go**: `io.Writer.Write()` 返回写入的字节数和错误

## 管道操作（Piping）

### 基本管道

```javascript
const fs = require('fs');

// 文件复制
const readableStream = fs.createReadStream('input.txt');
const writableStream = fs.createWriteStream('output.txt');

readableStream.pipe(writableStream);

readableStream.on('end', () => {
  console.log('文件复制完成');
});

// 错误处理
readableStream.on('error', (err) => {
  console.error('读取错误:', err);
  writableStream.destroy();
});

writableStream.on('error', (err) => {
  console.error('写入错误:', err);
  readableStream.destroy();
});
```

### 链式管道

```javascript
const fs = require('fs');
const zlib = require('zlib');

// 读取文件 -> 压缩 -> 写入文件
fs.createReadStream('large-file.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('large-file.txt.gz'))
  .on('finish', () => {
    console.log('文件压缩完成');
  });
```

**与其他语言对比**：
- **Python**: `shutil.copyfileobj(src, dst)` 实现管道
- **Java**: `Files.copy()` 或手动循环读取写入
- **Go**: `io.Copy(dst, src)` 实现管道

## 转换流（Transform Stream）

### 创建自定义转换流

```javascript
const { Transform } = require('stream');

// 创建大写转换流
class UpperCaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    // 将数据转换为大写
    const upperChunk = chunk.toString().toUpperCase();
    this.push(upperChunk);
    callback();
  }
}

// 使用转换流
const fs = require('fs');

fs.createReadStream('input.txt')
  .pipe(new UpperCaseTransform())
  .pipe(fs.createWriteStream('output.txt'))
  .on('finish', () => {
    console.log('转换完成');
  });
```

### 实际应用：JSON 解析流

```javascript
const { Transform } = require('stream');

class JSONParseTransform extends Transform {
  constructor() {
    super({ objectMode: true });  // 对象模式
    this.buffer = '';
  }
  
  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    
    // 尝试解析完整的 JSON 对象
    let start = 0;
    let depth = 0;
    
    for (let i = 0; i < this.buffer.length; i++) {
      if (this.buffer[i] === '{') depth++;
      if (this.buffer[i] === '}') {
        depth--;
        if (depth === 0) {
          try {
            const json = JSON.parse(this.buffer.slice(start, i + 1));
            this.push(json);
            start = i + 1;
          } catch (err) {
            return callback(err);
          }
        }
      }
    }
    
    this.buffer = this.buffer.slice(start);
    callback();
  }
}

// 使用示例
const fs = require('fs');

fs.createReadStream('data.jsonl')
  .pipe(new JSONParseTransform())
  .on('data', (obj) => {
    console.log('解析的 JSON 对象:', obj);
  });
```

## 双工流（Duplex Stream）

### 创建自定义双工流

```javascript
const { Duplex } = require('stream');

class EchoDuplex extends Duplex {
  constructor() {
    super();
    this.buffer = [];
  }
  
  _read(size) {
    // 从缓冲区读取数据
    while (this.buffer.length > 0) {
      const chunk = this.buffer.shift();
      if (!this.push(chunk)) {
        break;  // 如果 push 返回 false，停止推送
      }
    }
  }
  
  _write(chunk, encoding, callback) {
    // 写入数据，同时回显
    console.log('接收到:', chunk.toString());
    this.buffer.push(chunk);
    callback();
  }
}

// 使用示例
const echo = new EchoDuplex();

echo.write('Hello');
echo.write(' World');

echo.on('data', (chunk) => {
  console.log('读取到:', chunk.toString());
});
```

## 实际应用场景

### 场景 1：大文件处理

```javascript
const fs = require('fs');
const crypto = require('crypto');

// 计算大文件的 SHA256 哈希值
function calculateFileHash(filePath) {
  return new Promise((resolve, reject) => {
    const hash = crypto.createHash('sha256');
    const stream = fs.createReadStream(filePath);
    
    stream.on('data', (chunk) => {
      hash.update(chunk);
    });
    
    stream.on('end', () => {
      resolve(hash.digest('hex'));
    });
    
    stream.on('error', reject);
  });
}

// 使用
calculateFileHash('large-file.zip')
  .then(hash => {
    console.log('文件哈希值:', hash);
  })
  .catch(err => {
    console.error('计算失败:', err);
  });
```

### 场景 2：HTTP 响应流

```javascript
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  // 流式传输大文件
  const fileStream = fs.createReadStream('large-video.mp4');
  
  // 设置响应头
  res.setHeader('Content-Type', 'video/mp4');
  res.setHeader('Content-Length', fs.statSync('large-video.mp4').size);
  
  // 管道传输
  fileStream.pipe(res);
  
  fileStream.on('error', (err) => {
    console.error('文件读取错误:', err);
    if (!res.headersSent) {
      res.statusCode = 500;
      res.end('文件读取失败');
    }
  });
});

server.listen(3000, () => {
  console.log('服务器运行在 http://localhost:3000');
});
```

### 场景 3：数据转换管道

```javascript
const { Transform } = require('stream');
const fs = require('fs');

// CSV 转 JSON 转换流
class CSVToJSONTransform extends Transform {
  constructor() {
    super({ objectMode: true });
    this.headers = null;
    this.buffer = '';
  }
  
  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    
    // 保留最后一行（可能不完整）
    this.buffer = lines.pop() || '';
    
    for (const line of lines) {
      if (!line.trim()) continue;
      
      const values = line.split(',');
      
      if (!this.headers) {
        this.headers = values;
      } else {
        const obj = {};
        this.headers.forEach((header, index) => {
          obj[header.trim()] = values[index]?.trim() || '';
        });
        this.push(JSON.stringify(obj) + '\n');
      }
    }
    
    callback();
  }
  
  _flush(callback) {
    // 处理剩余数据
    if (this.buffer && this.headers) {
      const values = this.buffer.split(',');
      const obj = {};
      this.headers.forEach((header, index) => {
        obj[header.trim()] = values[index]?.trim() || '';
      });
      this.push(JSON.stringify(obj) + '\n');
    }
    callback();
  }
}

// 使用
fs.createReadStream('data.csv')
  .pipe(new CSVToJSONTransform())
  .pipe(fs.createWriteStream('data.jsonl'));
```

### 场景 4：实时日志处理

```javascript
const { Transform } = require('stream');
const fs = require('fs');

// 日志过滤和格式化转换流
class LogFilterTransform extends Transform {
  constructor(filterLevel = 'INFO') {
    super();
    this.filterLevel = filterLevel;
    this.levels = { DEBUG: 0, INFO: 1, WARN: 2, ERROR: 3 };
  }
  
  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');
    
    for (const line of lines) {
      if (!line.trim()) continue;
      
      // 解析日志级别
      const match = line.match(/\[(\w+)\]/);
      if (match) {
        const level = match[1];
        if (this.levels[level] >= this.levels[this.filterLevel]) {
          // 格式化输出
          const formatted = `[${new Date().toISOString()}] ${line}\n`;
          this.push(formatted);
        }
      }
    }
    
    callback();
  }
}

// 使用：监控日志文件
const tail = require('tail').Tail;
const logFile = new tail('app.log');

logFile.on('line', (line) => {
  // 通过转换流处理
});

// 或者使用 fs.watch
fs.watchFile('app.log', { interval: 1000 }, () => {
  fs.createReadStream('app.log', { start: fs.statSync('app.log').size })
    .pipe(new LogFilterTransform('WARN'))
    .pipe(process.stdout);
});
```

### 场景 5：流式数据处理

```javascript
const { Readable, Transform } = require('stream');

// 创建可读流（从数组生成数据）
class ArrayReadable extends Readable {
  constructor(array) {
    super({ objectMode: true });
    this.array = array;
    this.index = 0;
  }
  
  _read() {
    if (this.index < this.array.length) {
      this.push(this.array[this.index++]);
    } else {
      this.push(null);  // 结束流
    }
  }
}

// 数据处理转换流
class DataProcessor extends Transform {
  constructor() {
    super({ objectMode: true });
  }
  
  _transform(data, encoding, callback) {
    // 处理数据
    const processed = {
      ...data,
      processed: true,
      timestamp: Date.now()
    };
    this.push(processed);
    callback();
  }
}

// 使用
const data = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
  { id: 3, name: 'Charlie' }
];

new ArrayReadable(data)
  .pipe(new DataProcessor())
  .on('data', (item) => {
    console.log('处理后的数据:', item);
  });
```

## 最佳实践

### 1. 错误处理

```javascript
const fs = require('fs');

const readableStream = fs.createReadStream('input.txt');
const writableStream = fs.createWriteStream('output.txt');

readableStream
  .pipe(writableStream)
  .on('error', (err) => {
    console.error('管道错误:', err);
    // 清理资源
    readableStream.destroy();
    writableStream.destroy();
  });
```

### 2. 使用 pipeline（Node.js 10+）

```javascript
const { pipeline } = require('stream');
const fs = require('fs');
const zlib = require('zlib');

// pipeline 自动处理错误和清理
pipeline(
  fs.createReadStream('input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('output.txt.gz'),
  (err) => {
    if (err) {
      console.error('管道失败:', err);
    } else {
      console.log('管道成功完成');
    }
  }
);
```

### 3. 对象模式

```javascript
const { Transform } = require('stream');

// 对象模式允许传递 JavaScript 对象而不是 Buffer
class ObjectTransform extends Transform {
  constructor() {
    super({ objectMode: true });  // 启用对象模式
  }
  
  _transform(obj, encoding, callback) {
    // obj 是 JavaScript 对象，不是 Buffer
    this.push({ ...obj, processed: true });
    callback();
  }
}
```

### 4. 背压处理

```javascript
const writableStream = fs.createWriteStream('output.txt');

function writeData(data) {
  if (!writableStream.write(data)) {
    // 等待 drain 事件
    return new Promise((resolve) => {
      writableStream.once('drain', resolve);
    });
  }
  return Promise.resolve();
}

// 使用
async function writeLargeData() {
  for (let i = 0; i < 1000000; i++) {
    await writeData(`数据行 ${i}\n`);
  }
  writableStream.end();
}
```

## 总结

- **流**是 Node.js 处理大文件和实时数据的核心机制
- 四种流类型：Readable、Writable、Duplex、Transform
- 使用 **pipe()** 或 **pipeline()** 连接流
- 正确处理**背压**，避免内存溢出
- 始终进行**错误处理**，使用 pipeline 自动清理
- **对象模式**适合处理结构化数据
- 流式处理比一次性加载更**高效**，适合大文件

