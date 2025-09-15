# Zig WebAssembly Demo (wasm32-freestanding)

本项目展示如何用 Zig 生成可在浏览器中调用的 WebAssembly 模块，并提供然后访问：`http://localhost:8000/index.html`

页面应显示：`add(20,22) = 42`，说明 WebAssembly 模块加载和函数调用成功。# 目录结构

```
wasmdemo/
├─ src/
│  └─ lib.zig        # 导出函数示例（add）
├─ index.html        # 浏览器端加载和调用 wasm 的示例
├─ lib.wasm          # 由命令生成的 wasm 模块（运行构建命令后出现）
└─ README.md
```

## 先决条件
- 已安装 Zig（建议较新版本）。
- Windows PowerShell（或任意终端）。
- 浏览器端建议通过本地 HTTP 服务器打开页面，以避免 `instantiateStreaming` 在 file:// 下的 MIME/跨域问题。

## 快速开始：构建 WebAssembly 模块

### 1. 源代码准备
确保 `src/lib.zig` 包含导出函数：
```zig
pub export fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

### 2. 构建 .wasm 模块
在项目根目录 `F:\Demo\zig\wasmdemo` 执行以下命令：

```powershell
# 核心构建命令：生成浏览器可用的 wasm 模块
zig build-exe -target wasm32-freestanding -O ReleaseSmall -fno-entry .\src\lib.zig
```

**命令说明**：
- `-target wasm32-freestanding`：目标平台为独立的 WebAssembly（适合浏览器）
- `-O ReleaseSmall`：小体积优化构建
- `-fno-entry`：无入口点，纯导出函数模式（浏览器实例化后调用导出函数）
- `.\src\lib.zig`：源文件路径

**构建结果**：成功后会在当前目录生成 `lib.wasm` 文件（约 36 字节）。

> 说明：`zig build-lib` 默认产出静态库（.a），不适合作为浏览器直接加载的 wasm；而 `build-exe -fno-entry` 会链接成一个仅包含导出函数、没有 `_start` 的 wasm 模块，方便在浏览器中使用。

### 3. 验证构建结果
检查生成的文件：
```powershell
Get-Item .\lib.wasm | Format-List Name,Length,LastWriteTime
```

如果成功，应显示类似：
```
Name          : lib.wasm
Length        : 36
LastWriteTime : [当前时间]
```

## 在浏览器中加载并调用

项目已附带 `index.html`，其核心逻辑如下：
```html
<script type="module">
  async function main() {
    const res = await fetch('./lib.wasm');
    const { instance } = await WebAssembly.instantiateStreaming(res);
    const sum = instance.exports.add(20, 22);
    document.getElementById('out').textContent = `add(20,22) = ${sum}`;
  }
  main();
</script>
```

### 4. 启动本地 HTTP 服务器并测试

使用项目自带的 HTTP 服务器：
```powershell
# 启动本地服务器（监听 8080 端口）
.\dhgohttp.exe
```

然后在浏览器访问：`http://localhost:8080/index.html`

## 下一步（可选）
- 添加 `build.zig`，用 `zig build -Dtarget=wasm32-freestanding -Doptimize=ReleaseSmall` 一键构建并把产物安装到 `zig-out`。
- 扩展导出 API，演示内存交互（传递字符串、数组、内存布局）。
