# Zig WebAssembly Demo (wasm32-freestanding)

本项目展示如何用 Zig 生成可在浏览器中调用的 WebAssembly 模块，并提供最小示例页面调用导出函数。

## 目录结构

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

## 生成浏览器可用的 .wasm

对于浏览器/自定义宿主的使用场景，推荐使用无入口点（无 `_start`）的 wasm 模块，并通过 `pub export` 导出函数供宿主调用。

源代码（`src/lib.zig`）：
```zig
pub export fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

构建命令（在项目根目录执行）：
```powershell
# 生成无入口点的 wasm 模块（适合浏览器直接实例化并调用导出函数）
zig build-exe -target wasm32-freestanding -O ReleaseSmall -fno-entry .\src\lib.zig
# 生成后会在当前目录出现 lib.wasm
```

> 说明：`zig build-lib` 默认产出静态库（.a），不适合作为浏览器直接加载的 wasm；而 `build-exe -fno-entry` 会链接成一个仅包含导出函数、没有 `_start` 的 wasm 模块，方便在浏览器中使用。

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

建议使用本地 HTTP 服务器来打开页面（任选其一方式）：

- Python 3：
```powershell
cd F:\Demo\zig\wasmdemo
python -m http.server 8000
```
然后访问：`http://localhost:8000/index.html`

- PowerShell 简易服务器（需要具备脚本权限）：
```powershell
cd F:\Demo\zig\wasmdemo
# 以下一行命令在某些环境不可用，可优先选用 Python 方案
Start-Process powershell -ArgumentList "-NoProfile -Command \"Add-Type -AssemblyName System.Net.HttpListener; $listener = New-Object System.Net.HttpListener; $listener.Prefixes.Add('http://localhost:8000/'); $listener.Start(); Write-Host 'Serving http://localhost:8000'; while ($listener.IsListening) { $ctx=$listener.GetContext(); $path=Join-Path (Get-Location) ($ctx.Request.Url.LocalPath.TrimStart('/')); if ([string]::IsNullOrEmpty($path) -or -not (Test-Path $path)) { $path='index.html' } $bytes=[System.IO.File]::ReadAllBytes($path); $ctx.Response.ContentType = if ($path.EndsWith('.wasm')) { 'application/wasm' } elseif ($path.EndsWith('.html')) { 'text/html' } else { 'application/octet-stream' }; $ctx.Response.OutputStream.Write($bytes,0,$bytes.Length); $ctx.Response.Close() }\""
```
然后访问：`http://localhost:8000/index.html`

## 常见问题

- 为什么我用 `zig build-lib` 得到的是 `.a`？
  - `build-lib` 默认产出静态库（.a）。对 `wasm32-freestanding` 也不会变为可直接加载的动态库；且该目标不支持 `-dynamic`。
- 我能否指定输出文件名？
  - 直接使用 `zig build-exe ...` 会生成 `lib.wasm`。若想更名，建议在构建完后重命名文件，或使用 `build.zig` 自定义安装名。
- WASI 场景该如何构建？
  - 如果要在 Wasmtime/Wasmer/Node(WASI) 中运行，使用 `wasm32-wasi` 目标，并提供入口：
```powershell
zig build-exe -target wasm32-wasi -O ReleaseSmall .\src\main.zig
```
  - 其中 `main.zig` 包含 `pub fn main() !void { ... }`。这种产物包含 `_start`，更像“可执行”。

## 下一步（可选）
- 添加 `build.zig`，用 `zig build -Dtarget=wasm32-freestanding -Doptimize=ReleaseSmall` 一键构建并把产物安装到 `zig-out`。
- 扩展导出 API，演示内存交互（传递字符串、数组、内存布局）。
