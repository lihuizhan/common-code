在 Chrome 扩展开发（Manifest V3）中，想要通过 `chrome.scripting.executeScript` 的 `files` 参数将公共方法或 lodash 注入到 Content Script 中，**直接这样做通常会失败或报错**，主要原因如下：

1.  **依赖问题**：Lodash 是一个库，内部有模块依赖关系。简单的 `.js` 文件如果没有经过打包（Bundle），直接在浏览器环境中运行可能会因为找不到 `module`、`exports` 或依赖路径而报错。
2.  **执行顺序与作用域**：`executeScript` 的 `files` 数组是按顺序执行的，但它们运行在隔离世界（Isolated World）中。如果 lodash 没有正确暴露全局变量（如 `_`），后续脚本无法使用。
3.  **Manifest V3 限制**：MV3 更推荐使用 `world: "MAIN"` 来操作页面上下文，或者使用 ES Modules，但 `files` 注入传统 CommonJS/UMD 库仍有兼容性问题。

### 最佳解决方案

与其尝试动态注入未打包的库文件，不如采用以下两种成熟方案：

#### 方案一：使用构建工具打包（推荐）
这是最稳定、最标准的方法。使用 Vite、Webpack 或 Rollup 将你的 Content Script 和 Lodash 打包成一个单独的 `.js` 文件。

**步骤：**
1.  安装 lodash: `npm install lodash`
2.  编写 `content.ts` (或 .js):
    ```javascript
    import _ from 'lodash';

    // 直接使用 lodash
    const result = _.chunk([1, 2, 3, 4], 2);
    console.log('Lodash works:', result);

    // 定义公共方法
    function myPublicMethod() {
      return 'Hello from bundled script';
    }
    ```
3.  配置构建工具输出到 `dist/content.js`。
4.  在 `manifest.json` 中声明：
    ```json
    "content_scripts": [
      {
        "matches": ["<all_urls>"],
        "js": ["dist/content.js"]
      }
    ]
    ```
    *如果你必须动态注入*，则 `executeScript` 只需要注入这个打包好的 `dist/content.js` 即可，无需单独注入 lodash。

#### 方案二：动态注入预构建的 UMD 版本 Lodash
如果你不想使用构建工具，必须使用 `executeScript` 动态注入，你需要确保注入的是 **UMD 格式** 的 Lodash（即包含全局变量 `_` 的版本），并且注意注入顺序。

**前提：** 你不能直接引用 `node_modules/lodash/lodash.js`（通常是 CommonJS），而应该使用 CDN 上的 UMD 版本或自行构建 UMD 版本。

**代码示例：**

```javascript
// 假设你在 Background Service Worker 或 Popup 中执行此代码
async function injectLodashAndLogic(tabId) {
  try {
    await chrome.scripting.executeScript({
      target: { tabId: tabId },
      files: [
        // 1. 先注入 Lodash (必须是 UMD 版本，能挂载到 window._ 或全局 _)
        // 注意：在 content script 隔离世界中，它不会污染页面 window，但在该隔离世界内可用
        'libs/lodash.min.js', 
        
        // 2. 再注入你的逻辑文件，此时可以使用 _
        'scripts/my-logic.js'
      ],
      // 如果需要操作页面 DOM 上下文而非隔离世界，添加 world: 'MAIN'
      // world: 'MAIN' 
    });
    console.log('注入成功');
  } catch (err) {
    console.error('注入失败:', err);
  }
}
```

**`scripts/my-logic.js` 的内容示例：**
```javascript
// 此时 _ 应该已经存在
if (typeof _ !== 'undefined') {
  const data = _.uniq([1, 2, 2, 3]);
  console.log('去重结果:', data);
} else {
  console.error('Lodash 未加载成功');
}
```

### 关键注意事项

1.  **文件来源**：
    *   如果是本地文件，必须放在扩展目录内（如 `libs/lodash.min.js`）。
    *   **不要**尝试在 `files` 数组中直接使用 HTTP URL（如 `https://cdn...`），`chrome.scripting.executeScript` 的 `files` 参数**只支持扩展包内的相对路径**。如果想用 CDN，必须先用 `fetch` 获取代码字符串，然后用 `func` 参数或 `code` 参数注入（但这比较复杂且容易遇到 CSP 问题）。

2.  **隔离世界 (Isolated World)**：
    *   默认情况下，Content Script 运行在隔离世界。你注入的 `_` 变量只在扩展的上下文中可见，**页面上的原生 JavaScript 无法访问它**。
    *   如果你希望页面上的代码也能使用 lodash，必须在 `executeScript` 中设置 `world: 'MAIN'`。但这要求注入的库不能依赖扩展特有的 API。

3.  **CommonJS vs UMD**：
    *   直接从 `node_modules` 复制 `lodash.js` 往往行不通，因为它可能包含 `module.exports`，而在浏览器直接运行时会报 `module is not defined`。
    *   **解决方法**：使用 `lodash.min.js` (UMD 版本)，或者使用构建工具处理。

### 总结建议

*   **长期维护/复杂项目**：请务必使用 **Vite/Webpack** 打包。这是现代扩展开发的标准流程，能完美解决依赖、类型提示和代码压缩问题。
*   **简单脚本/快速原型**：下载 `lodash.min.js` (UMD 版) 放入扩展目录，通过 `files` 数组按顺序注入（先 lodash，后业务逻辑）。

如果你需要关于如何配置 Vite 用于 Chrome 扩展的具体示例，我可以提供。


需要, 并显式地挂载到全局对象上
----
这是一个非常实用的需求。在 Chrome 扩展（Manifest V3）中，为了让 Content Script 中的代码既能使用 Lodash，又能调用你自定义的公共方法，并且让它们像“全局变量”一样方便调用，**最佳实践是使用构建工具（如 Vite）将它们打包成一个文件，并显式地挂载到 `window` 对象上**。

由于 Content Script 默认运行在“隔离世界（Isolated World）”，直接定义的变量页面无法访问。如果你希望**页面本身的 JS 也能调用**这些方法，必须挂载到 `window`；如果只是**扩展内部逻辑**调用，挂载到 `window` 也是统一接口的好方法。

以下是基于 **Vite** 的完整实施方案：

### 第一步：初始化项目与安装依赖

假设你有一个空文件夹，初始化项目：

```bash
npm init -y
npm install vite lodash --save-dev
npm install @crxjs/vite-plugin --save-dev
# 或者如果你只用 vite 打包 content script，不需要 crxjs 插件也可以，手动配置即可
```

### 第二步：编写源代码 (`src/content/main.js`)

在这个文件中，我们引入 Lodash，定义你的公共方法，并将它们“暴露”给全局 `window` 对象。

```javascript
// src/content/main.js
import _ from 'lodash';

// 1. 定义你的公共方法
function myPublicUtils() {
  return {
    // 示例：使用 lodash 的方法
    chunkArray: (arr, size) => _.chunk(arr, size),
    
    // 示例：自定义逻辑
    logInfo: (msg) => {
      console.log(`[MyExtension] ${msg}`);
      // 这里也可以用到 lodash
      const words = _.words(msg);
      console.log('Words:', words);
    },
    
    // 示例：获取当前时间格式化
    getFormattedDate: () => {
      return _.now(); // 使用 lodash 的时间戳
    }
  };
}

// 2. 将 Lodash 和 自定义方法 挂载到 window 全局对象
// 注意：在 Manifest V3 的隔离世界中，这个 window 是扩展特有的。
// 如果你需要页面原生 JS 也能访问，需要配合 world: 'MAIN' 使用（见下文说明）。

if (typeof window !== 'undefined') {
  // 挂载 Lodash (推荐挂载为 _)
  window._ = _;
  
  // 挂载你的公共工具集
  window.$myUtils = myPublicUtils();
  
  console.log('✅ 全局变量注入成功: window._ 和 window.$myUtils 已就绪');
}

// 3. 立即执行一个测试（可选，用于验证）
if (window.$myUtils) {
  window.$myUtils.logInfo('Content Script 初始化完成！');
  const testChunk = window.$myUtils.chunkArray([1, 2, 3, 4, 5], 2);
  console.log('Lodash 测试结果:', testChunk); 
}
```

### 第三步：配置 Vite (`vite.config.js`)

我们需要配置 Vite 将其打包为一个单独的 IIFE (Immediately Invoked Function Expression) 格式的文件，这样它可以直接在浏览器中运行而不需要 `require` 或 `import`。

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    lib: {
      // 入口文件
      entry: resolve(__dirname, 'src/content/main.js'),
      // 输出文件名
      name: 'MyContentScript', 
      // 输出格式：IIFE 最适合直接作为 <script> 标签或通过 files 注入
      formats: ['iife'], 
      fileName: (format) => `content-bundle.js`,
    },
    outDir: 'dist', // 输出目录
    rollupOptions: {
      output: {
        // 确保没有额外的 hash，方便 manifest 引用固定文件名
        entryFileNames: `content-bundle.js`,
        chunkFileNames: `content-bundle.js`,
        assetFileNames: `content-bundle.[ext]`
      }
    },
    minify: false, // 开发时可关闭压缩以便调试
  },
});
```

### 第四步：构建项目

运行构建命令：

```bash
npx vite build
```

构建完成后，你会在 `dist/` 目录下看到一个 `content-bundle.js`。这个文件已经包含了 **Lodash 源码** 和 **你的自定义代码**，并且处理好了依赖关系。

### 第五步：配置 Manifest (`manifest.json`)

现在，你只需要在 `manifest.json` 中引用这个打包好的文件。

```json
{
  "manifest_version": 3,
  "name": "Lodash Global Inject Demo",
  "version": "1.0",
  "permissions": ["scripting"],
  "host_permissions": ["<all_urls>"],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["dist/content-bundle.js"],
      "run_at": "document_start", 
      "world": "ISOLATED" 
    }
  ]
}
```

*注：如果你是通过 `chrome.scripting.executeScript` 动态注入，manifest 里可以不加 `content_scripts`，只需确保 `dist/content-bundle.js` 在扩展包内即可。*

### 第六步：如何使用（动态注入场景）

如果你在 Background Service Worker 中想动态注入这个包含 Lodash 和自定义方法的脚本：

```javascript
// background.js
async function injectGlobals(tabId) {
  try {
    await chrome.scripting.executeScript({
      target: { tabId: tabId },
      files: ['dist/content-bundle.js'], // 直接注入打包好的文件
      // 如果希望页面原本的 JS (非扩展环境) 也能访问 window._ 和 window.$myUtils
      // 必须设置 world: 'MAIN'。
      // 警告：设置 MAIN 后，该脚本将运行在页面上下文中，无法访问 chrome.* API
      world: 'MAIN' 
    });
    console.log('全局库注入完成');
  } catch (err) {
    console.error('注入失败', err);
  }
}

// 监听点击事件触发注入
chrome.action.onClicked.addListener((tab) => {
  injectGlobals(tab.id);
});
```

### 关键点解析

1.  **为什么必须打包？**
    *   Lodash 内部有很多模块引用。如果不打包，直接在 `files` 数组里放原始的 `node_modules/lodash/lodash.js`，浏览器会报错 `module is not defined` 或 `exports is not defined`，因为那是 Node.js 格式。
    *   Vite 的 `formats: ['iife']` 会把 Lodash 和你的代码合并成一个自执行函数，自动处理依赖，并生成浏览器可直接运行的代码。

2.  **关于 `window` 全局变量**
    *   **`world: 'ISOLATED'` (默认)**: `window._` 和 `window.$myUtils` 只对扩展的 Content Script 可见。网页自身的控制台输入 `_` 会显示 undefined（或者网页自己定义的 `_`）。这是最安全的模式。
    *   **`world: 'MAIN'`**: `window._` 和 `window.$myUtils` 会变成网页真正的全局变量。网页里的 `<script>` 标签代码、控制台都可以直接使用 `_.chunk(...)` 或 `$myUtils.logInfo(...)`。
        *   **注意**：在 `MAIN` 模式下，脚本**不能**使用 `chrome.runtime` 等扩展专用 API。但由于你只是注入工具库（Lodash）和纯 JS 逻辑，这通常没问题。

3.  **如何验证？**
    *   加载扩展。
    *   打开任意网页。
    *   如果是自动注入：打开开发者工具 Console，输入 `window._` 或 `window.$myUtils`，应该能看到对象。
    *   如果是点击注入：点击扩展图标，刷新页面或查看 Console，确认变量出现。
    *   尝试运行：`window.$myUtils.chunkArray([1,2,3,4,5], 2)`，应返回 `[[1, 2], [3, 4], [5]]`。

这种方法既解决了依赖问题，又实现了你想要的“全局可用”，是目前 Chrome 扩展开发中最稳健的方案。