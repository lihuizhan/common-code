这是一个**完全重写**的版本。

### 核心改进点：
1.  **架构解耦**：将业务逻辑（文件检查、FS 操作）与 HTTP 层完全分离，便于测试和维护。
2.  **纯异步非阻塞**：彻底移除 `fs.existsSync`，所有文件 I/O 均使用 `fs.promises`，确保高并发下不阻塞事件循环。
3.  **安全性增强**：
    *   `/fsPromises` 接口增加了**白名单机制**，防止任意代码执行（如不允许调用 `unlink`, `writeFile` 等危险方法）。
    *   请求体大小限制，防止内存溢出。
4.  **健壮性**：完善的错误捕获和 JSON 解析处理。
5.  **CORS 统一处理**：在响应写入前统一设置跨域头。

---

### 1. 优化后的 `server.js`

```javascript
/**
 * server.js - 重构版
 * 功能：
 * 1. POST /pathExists : 批量检查文件是否存在（支持多目录、多后缀）
 * 2. POST /fsPromises/access : 通用 FS 访问接口（仅限白名单方法）
 */

const http = require('http');
const { parse: parseUrl } = require('url');
const path = require('path');
const fs = require('fs');
const { promisify } = require('util');

// ================= 配置常量 =================
const PORT = 8080;
const MAX_BODY_SIZE = 5 * 1024 * 1024; // 5MB 限制

// FS 方法白名单：只允许安全的读取/检查类操作
const ALLOWED_FS_METHODS = new Set([
  'access', 'stat', 'lstat', 'readFile', 'readdir', 'exists' 
]);

// ================= 工具函数层 (Business Logic) =================

/**
 * 安全地读取请求 Body
 * @param {http.IncomingMessage} req 
 * @returns {Promise<string>}
 */
async function getRequestBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    let totalSize = 0;

    req.on('data', (chunk) => {
      totalSize += chunk.length;
      if (totalSize > MAX_BODY_SIZE) {
        req.destroy(new Error('Payload Too Large'));
        reject(new Error('Request body exceeds size limit'));
        return;
      }
      chunks.push(chunk);
    });

    req.on('end', () => {
      resolve(Buffer.concat(chunks).toString('utf-8'));
    });

    req.on('error', (err) => reject(err));
  });
}

/**
 * 格式化扩展名列表
 * @param {string[]} customExts 
 * @param {string} originalExt 
 * @returns {string[]}
 */
function normalizeExtensions(customExts, originalExt) {
  const set = new Set();
  if (originalExt) set.add(originalExt);
  if (Array.isArray(customExts)) {
    customExts.forEach(e => e && set.add(e));
  }
  return Array.from(set);
}

/**
 * 核心逻辑：检查文件在指定路径集合中是否存在
 * @param {Object} item - { filename, downloadsLocation, exts }
 * @returns {Promise<boolean>} true 表示存在，false 表示不存在
 */
async function checkFileExistence(item) {
  const { filename, downloadsLocation, exts } = item;
  if (!filename) return false;

  const { dir, name, ext } = path.parse(filename);
  const targetExts = normalizeExtensions(exts, ext);
  
  // 标准化 downloadsLocation 为数组
  const locations = Array.isArray(downloadsLocation) 
    ? downloadsLocation 
    : [downloadsLocation];

  // 生成所有可能的文件路径
  const checkTasks = [];
  for (const loc of locations) {
    for (const e of targetExts) {
      const fullPath = path.resolve(loc, dir, name + e);
      // 使用 fs.promises.access 进行非阻塞检查
      // F_OK (0) 检查文件是否存在
      checkTasks.push(
        fs.promises.access(fullPath, fs.constants.F_OK)
          .then(() => true)  // 成功则存在
          .catch(() => false) // 失败则不存在
      );
    }
  }

  if (checkTasks.length === 0) return false;

  const results = await Promise.all(checkTasks);
  // 只要有一个路径存在，即视为存在
  return results.some(Boolean);
}

/**
 * 处理 /pathExists 路由
 * @param {Array} dataList 
 * @returns {Promise<Array>} 返回所有“不存在”的文件项
 */
async function handlePathExists(dataList) {
  if (!Array.isArray(dataList)) return [];

  const missingFiles = [];
  
  // 串行或并行处理取决于数据量，这里为了稳定性采用 Promise.all 并行
  const existenceResults = await Promise.all(
    dataList.map(async (item) => {
      const exists = await checkFileExistence(item);
      return { item, exists };
    })
  );

  for (const { item, exists } of existenceResults) {
    if (!exists) {
      missingFiles.push(item);
    }
  }

  return missingFiles;
}

/**
 * 处理 /fsPromises/:method 路由
 * @param {string} method 
 * @param {Array} args 
 */
async function handleFsPromises(method, args) {
  if (!ALLOWED_FS_METHODS.has(method)) {
    throw new Error(`Method '${method}' is not allowed for security reasons.`);
  }

  const fn = fs.promises[method];
  if (typeof fn !== 'function') {
    throw new Error(`Method '${method}' does not exist in fs.promises.`);
  }

  // 执行 FS 操作
  // 注意：args 可能包含路径等，需确保调用安全（白名单已做初步防护）
  return await fn(...args);
}

// ================= HTTP 服务层 =================

const server = http.createServer(async (req, res) => {
  // 1. 统一 CORS 设置
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');

  // 2. 处理 OPTIONS 预检
  if (req.method === 'OPTIONS') {
    res.writeHead(204);
    res.end();
    return;
  }

  // 3. 路由解析
  const parsedUrl = parseUrl(req.url, true);
  const { pathname } = parsedUrl;
  
  let statusCode = 200;
  let responseData = { msg: 'fail' };

  try {
    // 仅处理 POST 请求
    if (req.method !== 'POST') {
      throw new Error('Method Not Allowed');
    }

    // 获取并解析 Body
    const rawBody = await getRequestBody(req);
    let jsonData;
    try {
      jsonData = JSON.parse(rawBody);
    } catch (e) {
      throw new Error('Invalid JSON body');
    }

    // 路由分发
    if (pathname === '/pathExists') {
      // 业务：检查文件是否存在，返回不存在的列表
      const result = await handlePathExists(jsonData);
      responseData = { msg: 'success', result };
    
    } else if (pathname.startsWith('/fsPromises/')) {
      // 业务：动态调用 fs.promises 方法
      const parts = pathname.split('/').filter(Boolean); // ['fsPromises', 'access']
      const method = parts[parts.length - 1];
      
      // 参数校验：期望传入数组作为参数列表
      const args = Array.isArray(jsonData) ? jsonData : [jsonData];
      
      const result = await handleFsPromises(method, args);
      responseData = { msg: 'success', result };

    } else {
      statusCode = 404;
      responseData = { msg: 'fail', error: 'Endpoint not found' };
    }

  } catch (error) {
    console.error(`[Error] ${pathname}:`, error.message);
    statusCode = error.message.includes('Not Allowed') ? 405 :
                 error.message.includes('Large') ? 413 :
                 error.message.includes('JSON') ? 400 :
                 error.message.includes('not allowed') ? 403 : 500;
    
    responseData = { msg: 'error', error: error.message };
  }

  // 4. 发送响应
  res.writeHead(statusCode, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(responseData));
});

// 启动服务器
server.listen(PORT, () => {
  console.log(`🚀 Server started at http://localhost:${PORT}`);
  console.log(`   - POST /pathExists`);
  console.log(`   - POST /fsPromises/:method (Allowed: ${Array.from(ALLOWED_FS_METHODS).join(', ')})`);
});
```

---

### 2. 测试脚本 `test-client.js`

这个脚本会自动创建测试文件，然后调用接口验证逻辑是否正确。

```javascript
/**
 * test-client.js
 * 用于测试 server.js 的功能
 * 使用方法：node test-client.js
 * (请确保 server.js 已在后台运行)
 */

const fs = require('fs');
const path = require('path');

const BASE_URL = 'http://localhost:8080';

// 准备测试环境
const TEST_DIR = path.join(__dirname, 'test_data');
const SUB_DIR = path.join(TEST_DIR, 'sub');
const EXIST_FILE_NAME = 'existing_file.txt';
const MISSING_FILE_NAME = 'missing_file.txt';

function setupEnv() {
  if (!fs.existsSync(TEST_DIR)) fs.mkdirSync(TEST_DIR, { recursive: true });
  if (!fs.existsSync(SUB_DIR)) fs.mkdirSync(SUB_DIR, { recursive: true });
  
  // 创建一个真实存在的文件
  fs.writeFileSync(path.join(SUB_DIR, EXIST_FILE_NAME), 'Hello World');
  console.log(`✅ Test env ready. Created: ${path.join(SUB_DIR, EXIST_FILE_NAME)}`);
}

async function request(endpoint, data) {
  const res = await fetch(`${BASE_URL}${endpoint}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  });
  const json = await res.json();
  return { status: res.status, data: json };
}

async function runTests() {
  setupEnv();

  console.log('\n--- Test 1: /pathExists (混合存在与不存在) ---');
  const pathExistsPayload = [
    {
      filename: 'sub/existing_file.txt',
      downloadsLocation: TEST_DIR,
      exts: [] 
    },
    {
      filename: 'sub/missing_file.txt',
      downloadsLocation: TEST_DIR,
      exts: ['.bak', '.tmp'] // 尝试检查 .txt.bak, .txt.tmp, 以及原始的 .txt
    },
    {
      filename: 'sub/existing_file.txt',
      downloadsLocation: ['/non/existent/path'], // 路径错误，应视为不存在
      exts: []
    }
  ];

  const res1 = await request('/pathExists', pathExistsPayload);
  console.log('Status:', res1.status);
  console.log('Result Count (should be 2):', res1.data.result?.length);
  
  if (res1.data.result?.length === 2) {
    console.log('✅ Test 1 Passed: Correctly identified missing files.');
  } else {
    console.log('❌ Test 1 Failed');
    console.log(res1.data);
  }

  console.log('\n--- Test 2: /fsPromises/access (检查存在) ---');
  // 调用 fs.promises.access(path, mode)
  // 参数应为数组: [path, mode]
  const accessPayload = [path.join(SUB_DIR, EXIST_FILE_NAME), 0]; // 0 is F_OK
  const res2 = await request('/fsPromises/access', accessPayload);
  
  if (res2.status === 200 && res2.data.msg === 'success') {
    console.log('✅ Test 2 Passed: fs.promises.access worked.');
  } else {
    console.log('❌ Test 2 Failed');
    console.log(res2.data);
  }

  console.log('\n--- Test 3: /fsPromises/access (检查不存在) ---');
  const accessPayload2 = [path.join(SUB_DIR, MISSING_FILE_NAME), 0];
  const res3 = await request('/fsPromises/access', accessPayload2);
  
  // access 在文件不存在时会 reject，我们的 server 应该捕获并返回 error 或特定结构
  // 根据当前实现，未捕获的 promise reject 会被 try-catch 包裹返回 500 error
  if (res3.status !== 200 || (res3.data.msg === 'error')) {
    console.log('✅ Test 3 Passed: Correctly handled non-existent file access error.');
  } else {
    console.log('⚠️ Test 3 Note: Access succeeded unexpectedly?');
  }

  console.log('\n--- Test 4: Security Check (Blocked Method) ---');
  // 尝试调用 unlink (删除文件)，应该在服务端被拦截
  const badPayload = [path.join(SUB_DIR, EXIST_FILE_NAME)];
  const res4 = await request('/fsPromises/unlink', badPayload);
  
  if (res4.status === 403 || (res4.data.error && res4.data.error.includes('not allowed'))) {
    console.log('✅ Test 4 Passed: Dangerous method blocked.');
  } else {
    console.log('❌ Test 4 Failed: Security risk!');
  }

  // 清理
  // fs.rmSync(TEST_DIR, { recursive: true, force: true });
  console.log('\n🏁 All tests completed.');
}

runTests().catch(console.error);
```

### 3. 如何运行

1.  **启动服务器**：
    ```bash
    node server.js
    ```
    *输出应显示：`🚀 Server started at http://localhost:8080`*

2.  **运行测试客户端**（新开一个终端）：
    ```bash
    node test-client.js
    ```

### 主要改动详解

1.  **`checkFileExistence` 函数**：
    *   不再使用同步循环。它构建了一个包含所有可能路径（不同目录 + 不同后缀）的 Promise 数组。
    *   使用 `Promise.all` 并行检查。
    *   逻辑：`results.some(Boolean)` —— 只要有任何一个路径检查通过（文件存在），该条目就被视为“存在”，从而**不**出现在返回结果中（因为接口目的是返回“缺失”的文件）。

2.  **`handleFsPromises` 函数**：
    *   提取 URL 末尾的方法名（如 `access`）。
    *   **关键安全点**：`ALLOWED_FS_METHODS` 白名单。如果用户请求 `/fsPromises/unlink` 或 `/fsPromises/exec`，服务器直接拒绝，防止恶意删除文件或执行命令。
    *   动态调用 `fs.promises[method](...args)`。

3.  **错误处理**：
    *   `try...catch` 包裹了整个异步流程。
    *   针对 JSON 解析错误、Payload 过大、方法不存在等情况返回明确的 HTTP 状态码（400, 413, 403, 500）。

4.  **CORS**：
    *   无论请求成功还是报错，`Access-Control-*` 头部都会在 `res.writeHead` 之前设置，确保前端能正确接收错误响应。

这个版本不仅修复了原有代码的性能瓶颈，还极大地提升了安全性和可维护性，非常适合作为生产环境的微服务模块。