# Implant

考虑到红队人员的使用习惯， 本 `Implant` 所支持的命令将大量沿用 `CS` 工具的命令及使用习惯

欢迎各位对想要的功能和使用中遇到的问题提 `issues` 🙋

## Build

rust在编译上是个很复杂的语言.  malefic更是依赖了一些`nightly`的特性, 导致无法在所有rust版本上编译通过.  **需要指定特定日期版本的toolchain, target才能编译通过** . 

为此, 我们准备了多个编译方案, 有rust使用经验的用户可以[尝试使用本地环境编译](#_3) , 初次使用rust的用户建议通过[docker malefic-builder](#docker)进行. 

后续还将提供基于github action的自动化编译方案, 尽可能在编译上减少困难. 

### 环境准备

克隆[malefic](https://github.com/chainreactors/malefic)

```
git clone --recurse-submodules https://github.com/chainreactors/malefic
```

!!! tips "注意clone子项目"
	需要添加`--recurse-submodules`递归克隆子项目. 如果已经clone也不必担心,`git submodule update --init` 即可


#### 当前测试过支持的架构

malefic理论上支持rust能编译的所有平台, 包括各种冷门架构的IoT设备, Android系统, iOS系统等等 (有相关需求可以联系我们定制化适配)

当前支持的全部架构可参考[Targets](https://github.com/chainreactors/malefic/blob/malefic-builder/Makefile#L2).(欢迎提供反馈)

```
make windows_x64  # x86_64-pc-windows-gnu
make windows_x32  # i686-pc-windows-gnu
make linux_x64    # x86_64-unknown-linux-gnu
make linux_x32    # i686-unknown-linux-gnu
make darwin_x64   # x86_64-apple-darwin
make darwin_arm   # aarch64-apple-darwin
```

#### Docker编译环境

因为rust安装与编译的复杂性, 我们提供了 `Docker` 环境搭配makefile一键交叉编译malefic。


##### 获取编译环境

```bash
# 从ghcr获取编译镜像
docker pull ghcr.io/chainreactors/malefic-builder:v0.0.1-gnu
# 从dockerhub获取
docker pull chainreactors/malefic-builder:v0.0.1-gnu
```

!!! important "镜像略大"
	`malefic-builder:v0.0.1-gnu`镜像原始大小为3.1G,cr存储时使用了压缩算法把大小压缩到1G, 如果pull速度较慢推荐配置docker镜像源加速会快上不少

你也可以参考[从Dockerfile构建编译环境](#dockerfile)自行构建交叉编译环境.


##### 从Dockerfile构建编译环境

**在项目根目录(外层malefic)下执行docker build**

```
docker build -f builder/Dockerfile.GNU -t malefic-builder . 
```

!!! important "国内用户可通过编辑Dockerfile自行添加国内APT源"
	命令行执行:
	
	```
	sed -i '/RUN apt update/i RUN sed -i \"s|http://deb.debian.org/debian|http://mirrors.163.com/debian|g\" /etc/apt/sources.list' builder/Dockerfile.GNU
	```
	
	或手动添加 `builder/Dockerfile.GNU` 的 `RUN apt update` 前添加行
	```
	RUN sed -i "s|http://deb.debian.org/debian|http://mirrors.163.com/debian|g" /etc/apt/sources.list
	```

##### 运行docker容器

!!! important "请在项目根目录下运行"
	为了方便修改config以及编译参数, 我们选择了映射程序目录的方式实现

运行docker容器
```
# ghcr
docker run -v "$PWD/:/root/src" -it --name malefic-builder ghcr.io/chainreactors/malefic-builder:v0.0.1-gnu bash
# dockerhub
docker run -v "$PWD/:/root/src" -it --name malefic-builder chainreactors/malefic-builder:v0.0.1-gnu bash
```

#### Github Action编译环境
为了更方便的交叉编译，我们采用了[cross-rs/cross](https://github.com/cross-rs/cross/)的做法，并自行构建了对应的镜像用于适配malefic的编译，这些镜像已上传至[ghcr.io/chainreactors](https://github.com/orgs/chainreactors/packages)和
[chainreactors](https://hub.docker.com/u/chainreactors). 

首先使用gh登录github
```shell
# 交互式登录 github
gh auth login
# 或者使用token
windows: $ENV:GH_TOKEN="your_authentication"
linux: export GH_TOKEN="your_authentication"
```
修改完config.yaml配置后, 你可以通过gh来运行编译工作流，参考命令如下
```bash
gh workflow run generate.yml -f malefic_config=$(base64 </path/to/config.yaml>) -f remark="write somthing.." -f targets="x86_64-pc-windows-gnu,i686-pc-windows-gnu," -R <username/malefic>

```
查看编译进度
```bash
gh run list -R <username/malefic>
```
根据填写的remark和run_id，你可以很方便的找到对应的artifact下载(artifact默认保留时间为3天,可自行更改[retention-days](https://github.com/chainreactors/malefic/blob/master/.github/workflows/generate.yml#L90))
```bash
gh run download -R <username/malefic>
```
![gh-run-list-and-download](../assets/gh-run-list-download.png)

注意windows可能没有`base64`, 你可以通过`notepad $PROFILE`自定义一条函数
```powershell
function base64 {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline, ValueFromPipelineByPropertyName)]
        [string] $s,
        [switch] $decode,
        [switch] $binary
    )
    process {
        Set-StrictMode -Version Latest
        $ErrorActionPreference = 'Stop'
        if ($s.Length -le 320 -and (Test-Path $s -PathType Leaf)) {
            $str = Get-Content $s -AsByteStream
            $code = [System.Convert]::ToBase64String($str)
        }
        else {
            $code = [System.Convert]::ToBase64String([System.Text.Encoding]::utf8.GetBytes($s))
        }

        if ($decode) {
            if ($binary) {
                [System.Convert]::FromBase64String($s)
            }
            else {
                [System.Text.Encoding]::utf8.GetString( [System.Convert]::FromBase64String($s))
            }
        }
        else {
            $code
        }
    }
}
```



!!! danger "保护敏感信息"
    我们对config进行add-mask处理,保护config.yaml的敏感数据，但是github action输出的artifact或release仍会暴露, 使用时建议创建一份malefic到自己的仓库中设置为private再使用。

#### 本地编译环境

!!! danger "rust toolchain需要指定版本"
    我们使用了`nightly-2024-08-16`的一些特性， 因此需要特殊版本 `rust` 套件进行编译（现在已经通过`rust-toolchain.toml`自动配置toolchain， 如果没有进行过修改， 可以忽略这行）

我们推荐windows用户使用[msys2](https://www.msys2.org/)管理GNU工具链环境, 可通过官网二进制文件直接安装。

在msys2的terminal下执行如下安装可以保证64、32位GNU工具链的正常编译
```
pacman -Syy # 更新包列表
pacman -S --needed mingw-w64-x86_64-gcc
pacman -S --needed mingw-w64-i686-gcc
```

你可自行把msys64添加到环境变量中， 也可通过`notepad $PROFILE`将如下内容添加到powershell配置中，实现在powershell中快速切换`mingw64/32`.
```powershell
function mg {
    param (
        [ValidateSet("32", "64")]
        [string]$arch = "64"
    )
    
    $basePath = "D:\msys64\mingw" # 此处是你的msys2安装路径
    $env:PATH = "${basePath}${arch}\bin;" + $env:PATH
    Write-Host "Switched to mingw${arch} (bit) toolchain"
}
mg 64
```
用法参考下图:

![switch mingw](../assets/switch-mingw-in-powershell.png)

### Compile

#### 通过Makefile编译

不管在docker中, 还是本机中, 通过Malefile即可快速编译需要架构的二进制文件

!!! tips "windows安装make"
	windows可使用`scoop install make`或者`winget install make`安装Make工具

**build指定架构**
```bash
make windows_x64
```

(*[全部支持的架构](#_2)*)

**build 全部target的二进制文件**
```bash
make all
```

release文件将生成到对应 `target\[arch]\release\` 中, 也会映射到本机malefic项目中的`target\[arch]\release\`中

![win64-release](../assets/win64-release.png)


#### 手动编译malefic

makefile 提供了一些预设和编译优化选项. 熟悉rust的使用者也可以手动编译

添加对应的目标编译架构

```bash
rustup target add x86_64-pc-windows-gnu
```

指定target编译, 可以在[全部支持的架构](#_2)中找到对应的target

```bash
# mg 64
cargo build --release -p malefic --target x86_64-pc-windows-gnu
# mg 32
cargo build --release -p malefic --target i686-pc-windows-gnu
```


#### 编译独立modules

malefic的windows平台目前支持动态加载module, 因此可以编译单个或者一组module, 然后通过`load_module`给已上线的implant添加新的功能. 

[load_module使用文档](IoM/manual/help/#load_module)
[load_module相关介绍](#dynamic-module)

makefile指令如下

```bash
make profile_module FEATURES="sys_execute_shellcode sys_execute_assembly"
```

也可手动使用cargo编译

```bash
cargo build --release --features "sys_execute_shellcode sys_execute_assembly" -p malefic-modules --target x86_64-pc-windows-gnu
```

??? info "所有支持的feautres"
	请见 https://github.com/chainreactors/malefic/blob/master/malefic-modules/Cargo.toml
	
	fs_ls = ["fs"]  
	fs_cd = ["fs"]  
	fs_rm = ["fs"]  
	fs_cp = ["fs"]  
	fs_mv = ["fs"]  
	fs_pwd = ["fs"]  
	fs_mem = ["fs"]  
	fs_mkdir = ["fs"]  
	fs_chmod = ["fs"]  
	fs_cat = ["fs"]  
	  
	sys_info = ["sys"]  
	sys_ps = ["sys"]  
	sys_id = ["sys"]  
	sys_env = ["sys"]  
	sys_whoami = ["sys"]  
	sys_exec = ["sys"]  
	sys_kill = ["sys"]  
	sys_execute_shellcode = ["sys"]  
	sys_execute_assembly = ["sys"]  
	sys_execute_bof = ["sys"]  
	sys_execute_pe = ["sys"]  
	sys_execute_powershell = ["sys"]  
	sys_netstat = ["sys"]  
	  
	net_upload = ["net"]  
	net_download = ["net"]


编译结果为`target\[arch]\release\modules.dll`

可以使用`load_module`热加载这个dll 

!!! important "module动态加载目前只支持windows"
	linux与mac在理论上也可以实现

常见的使用场景:

1.  编译一个不带任何modules的malefic, 保持静态文件最小特征与最小体积. 通过`load_module modules.dll` 动态加载模块
2. 根据场景快速开发module, 然后动态加载到malefic中. 
3. 长时间保持静默的场景可以卸载所有的modules, 并进入到sleepmask的堆加密状态.  等需要操作时重新加载modules

## Config

`Implant` 拥有 `config.yaml` 以对生成的 `implant` 进行配置：

会在编译时通过`malefic-config` 自动解析各种feature与参数配置. 

### Server

与server通讯相关的配置. 

* `Server` 字段包含了以下连接配置:

	* `urls`: `implant` 所需要建立连接的目标 `ip:port` 或 `url:port` 列表
	
	* `protocol` : `implant` 所使用的传输协议
	
	* `tls` : `implant` 是否需要使用 `tls`
	
	* `interval` :  每次建立连接的时间间隔(单位为 `milliseconds`)
	
	* `jitter`: 每次建立连接时的时间间隔抖动(单位为 `milliseconds`)
	
	* `ca` : 所使用的证书路径

### implants

implant端各种opsec与高级特性的配置.  在community中带🔒表示配置不生效. 

`Implant` 字段包含以下可选生成物配置：

* `modules`: 生成物所需要包含的功能模块， 如默认提供的 `base` 基础模块及 `full` 全功能模块， 或自行组装所需功能模块, 详见章节 `Extension` 部分

#### metadata
* `metadata`: 生成物元特征：
    * `remap_path`: 编译绝对路径信息
    * `icon`
    * `file_version` 
    * `product_version`
    * `company_name`
    * `product_name`
    * `original_filename`
    * `file_description`
    * `internal_name`

#### apis 🔒

在 `EDR` 的对抗分析中， 我们支持在组装 `Implant` 时由用户自行选择使用各级别的 `API`， 如直接调用系统 `API`, 动态获取并调用， 通过 `sysall` 调用，这可以有效减少程序 `Import` 表所引入的的特征

在 `syscall` 调用中， 我们支持使用各类门技术来调用系统调用而非直接调用用户层 `API`， 以防止 `EDR` 对常用红队使用的 `API` 进行监控， 如何配置可见 `Implant Config File` 对应 `apis` 部分

* apis: 
    * `level` : 使用上层api还是nt api, `"sys_apis"` , `"nt_apis`
    * `priority`:
        * `normal` : 直接调用 
        * `dynamic` : 动态调用
            * `type`: 如自定义获取函数地址方法 `user_defined_dynamic`, 系统方法`sys_dynamic` (`LoadLibraryA/GetProcAddress`)
        * `syscall`: 通过 `syscall`调用
            * `type`: 生成方式, 函数式 `func_syscall`, inline 调用 `inline_syscall


#### alloctor 🔒
* allactor: 
    * `inprocess`: 进程内分配函数, `VirtualAlloc`, `VirtualAllocEx`, `HeapAlloc`, `NtAllocateVirtualMemory`, `VirtualAllocExNuma`, `NtMapViewOfSection`
    * `crossprocess`: 进程间分配函数, `VirtualAllocEx`, `NtAllocateVirtualMemory`,
    `VirtualAllocExNuma`, `NtMapViewOfSection`

#### advance feautres 🔒

`sleep_mask`: 睡眠混淆是否开启 👤

`sacriface_process`: 是否需要牺牲进程功能

`fork_and_run`: 是否需要使用 `fork and run` 机制

`hook_exit`: 是否需要对退出函数进行 `hook` 以防止误操作导致的退出

`thread_task_spoofer`: 是否需要自定义线程调用堆栈 👤

## Module

module是implant中功能的基本单元, 各种拓展能力(bof,pe,dll)的执行也依赖于module实现. 

### 已实现modules

不同操作系统与架构支持的module不同. 具体支持下表:

| 功能                | windows-x86 | windows-x86_64 | windows-arm* | linux-x86_64 | linux-arm | linux-aarch64 | macOS-intel | macOS-arm |
| ----------------- | ----------- | -------------- | ------------ | ------------ | --------- | ------------- | ----------- | --------- |
| ls                | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| cd                | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| mv                | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| pwd               | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| mem               | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| mkdir             | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| chomd             | ✗           | ✗              | ✗            | ✓            | ✓         | ✓             | ✓           | ✓         |
| chown             | ✗           | ✗              | ✗            | ✓            | ✓         | ✓             | ✓           | ✓         |
| cat               | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| upload            | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| download          | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| env               | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| kill              | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| whoami            | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| ps                | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| netstat           | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| exec              | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| command           | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| execute_shellcode | ✓           | ✓              | ✓            | ✓            | ✓         | ✓             | ✓           | ✓         |
| execute_assembly  | ✓           | ✓              | ✓            | ✗            | ✗         | ✗             | ✗           | ✗         |
| powershell        | ✓           | ✓              | ✓            | ✗            | ✗         | ✗             | ✗           | ✗         |
| execute_pe        | ✓           | ✓              | ✓            | ✗            | ✗         | ✗             | ✗           | ✗         |
| execute_bof       | ✓           | ✓              | ✓            | ✗            | ✗         | ✗             | ✗           | ✗         |
| hot_module_load   | ✓           | ✓              | ✓            | ✗            | ✗         | ✗             | ✗           | ✗         |


### Professional Features 🔒

部分module需要依赖各类kits中的高级特性, 在community中只提供了默认特征的版本.

| 目标系统 | 目标架构    | sleep_mask | obfstr | fork&run | thread_stack_spoof | syscall | dynamic_api |
| -------- | ----------- | ---------- | ------ | -------- | ------------------ | ------- | ----------- |
| windows  | x86         | ✗         | ✓     | ✓       | ✓                 | ✓      | ✓          |
|          | x86_64      | ✓         | ✓     | ✓       | ✓                 | ✓      | ✓          |
|          | arm/aarch64 | ✗         | ✓     | ✓       | ✓                 | ✗      | ✓          |
| linux    | intel       | ✗         | ✓     | ✗       | ✗                 | ✗      | ✗          |
|          | arm         | ✗         | ✓     | ✗       | ✗                 | ✗      | ✗          |
|          | mips        | ✗         | ✓     | ✗       | ✗                 | ✗      | ✗          |
| macOS    | intel       | ✗         | ✓     | ✗       | ✗                 | ✗      | ✗          |
|          | arm         | ✗         | ✓     | ✗       | ✗                 | ✗      | ✗          |

### Dynamic Module

malefic的设计理念之一就是模块化, 自由组装. modules部分的设计也提现了这个理念. 

通过rust自带的`features`相关功能, 可以控制编译过程中的模块组装.  目前提供了三种预设

??? info "modules预设"
```
full = [  
    "fs_ls",  
    "fs_cd",  
    "fs_rm",  
    "fs_cp",  
    "fs_mv",  
    "fs_pwd",  
    "fs_mem",  
    "fs_mkdir",  
    "fs_chmod",  
    "fs_cat",  
    "net_upload",  
    "net_download",  
    "sys_info",  
    "sys_exec",  
    "sys_execute_shellcode",  
    "sys_execute_assembly",  
    "sys_execute_powershell",  
    "sys_execute_bof",  
    "sys_execute_pe",  
    "sys_env",  
    "sys_kill",  
    "sys_whoami",  
    "sys_ps",  
    "sys_netstat",  
]  
  
base = [  
    "fs_ls",  
    "fs_cd",  
    "fs_rm",  
    "fs_cp",  
    "fs_mv",  
    "fs_pwd",  
    "fs_cat",  
    "net_upload",  
    "net_download",  
    "sys_exec",  
    "sys_env",  
]  
  
extend = [  
    "sys_kill",  
    "sys_whoami",  
    "sys_ps",  
    "sys_netstat",  
    "sys_execute_bof",  
    "sys_execute_shellcode",  
    "sys_execute_assembly",  
    "fs_mkdir",  
    "fs_chmod",  
]
```


当然也可以根据喜好自行组装功能模块， 当然， 我们也提供了动态加载及卸载模块的功能， 可以随时添加新模块.


!!! danger "编译时组装的模块无法被卸载" 
	这里有一个好消息与一个坏消息.
	坏消息是编译时组装的模块无法被卸载, 因此请根据自己的使用场景选择合适的预设.
	好消息是虽然无法卸载, 但加载新模块时如选用了同样名称的模块, 新模块将覆盖本体的模块.(在内存中原本的模块依旧会存在)

#### module定义

模块的开发者绝大多数场景下不需要关注除了`run`之外的方法. [开发自定义模块请见文档](IoM/manual/develop/#module)

```rust
#[async_trait]
pub trait Module {
    fn name() -> &'static str where Self: Sized;
    fn new() -> Self where Self: Sized;
    fn new_instance(&self) -> Box<MaleficModule>;
	async fn run(&mut self, 
				id: u32, 
				receiver: &mut crate::Input, 
				sender: &mut crate::Output) -> Result
```

#### module管理

就像开始所说的那样， `malefic` 支持编译时组装所需功能模块， 同时也支持启动后动态的加载和卸载所需的功能模块. 我们提供了一组api用来管理模块.  具体的使用请见[使用文档module部分](IoM/manual/help/#_2)

- `list_modules` 命令允许列举当前 `Implant` 所持有的模块
- `load_modules` 命令则支持动态加载本地新组装的模块， 只需要 `load_modules --name xxx --path module.dll` 即可动态加载新的模块， 请注意， 如本体已经含有的模块（生成时组装的模块）， 再次加载将会覆盖该模块的功能， 是的， `load_modules` 允许覆盖本体功能
- `unload_modules` 🛠️ 命令则会卸载使用 `load_modules` 命令所加载的对应 `name` 的模块， 请注意， 生成时确定的模块是无法卸载的， 但这些模块可以被加载的新模块所覆盖
- `refresh_modules` 🛠️ 命令将会卸载所有动态加载的模块， 包括覆盖掉的本体模块， 一切模块将恢复成编译时的初始状态

## Windows Kit

关于 `Windows` 平台特有功能， 可以查阅 [win_kit](implant_win_kit.md)