# LLVM Pass 插桩实现方法hook

在移动应用开发中，热修复技术是保障应用稳定性和快速迭代的重要手段。基于 LLVM Pass 的插桩技术，能够在编译阶段对代码进行修改，为热修复提供底层支持。本文将详细介绍如何搭建 LLVM 工程、创建并编译自定义 Pass，以及在实际项目中实现插桩的完整流程。

## LLVM 工程搭建
### 下载 LLVM 源码并构建 Xcode 工程

LLVM 工程的搭建是实现自定义 Pass 的基础，正确匹配 Xcode 版本和 LLVM 版本至关重要。

首先，需要确定与所使用 Xcode 对应的 LLVM 工具集版本。可以通过维基百科（[ht](https://en.wikipedia.org/wiki/Xcode)[tps:/](https://en.wikipedia.org/wiki/Xcode)[/en.w](https://en.wikipedia.org/wiki/Xcode)[ikipe](https://en.wikipedia.org/wiki/Xcode)[dia.o](https://en.wikipedia.org/wiki/Xcode)[rg/wi](https://en.wikipedia.org/wiki/Xcode)[ki/Xc](https://en.wikipedia.org/wiki/Xcode)[ode](https://en.wikipedia.org/wiki/Xcode)）查询，在页面中找到 Swift 版本对应的 LLVM 版本信息（看 swift 这一列的版本）。



![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/1bec69c5-7269-4317-907f-b8f7b1d6554b.png)

例如，Xcode 16.2 对应的 Swift 版本为 6.0.3，但如果 LLVM 没有完全匹配的版本，可选择最近的较小版本，如 6.0.2。

接下来，到 GitHub 的 Swift 官方 LLVM 仓库（[http](https://github.com/swiftlang/llvm-project/)[s://g](https://github.com/swiftlang/llvm-project/)[ithub](https://github.com/swiftlang/llvm-project/)[.com/](https://github.com/swiftlang/llvm-project/)[swift](https://github.com/swiftlang/llvm-project/)[la](https://github.com/swiftlang/llvm-project/)[ng/l](https://github.com/swiftlang/llvm-project/)[lvm-p](https://github.com/swiftlang/llvm-project/)[rojec](https://github.com/swiftlang/llvm-project/)[t/](https://github.com/swiftlang/llvm-project/)）下载与 Xcode 版本匹配的依赖 Swift 的 LLVM 版本源码。



![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/5259ae3b-b9a6-475b-96bc-ab5eace6e718.png)

下载完成后，进入源码目录进行工程构建。首先创建并进入构建目录：


```plaintext
cd llvm-project-swift-release-6.0.2
mkdir build
cd build

```

然后通过 CMake 生成 LLVM 的 Xcode 编译工程，在 build 目录下执行以下命令：

```shell
cmake -G Xcode \
      -DLLVM_ENABLE_PROJECTS="clang;lld" \
      -DLLVM_TARGETS_TO_BUILD="AArch64;X86" \
      -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_BUILD_EXAMPLES=ON \
      -DLLVM_INCLUDE_TESTS=OFF \
      -DLLVM_ENABLE_ASSERTIONS=OFF \
      -DLLVM_OPTIMIZED_TABLEGEN=ON \
      -DLLVM_USE_SPLIT_DWARF=ON \
      -DLLVM_PARALLEL_COMPILE_JOBS=8 \
      -DLLVM_PARALLEL_LINK_JOBS=4 \
      ../llvm
```

上述命令中，各参数具有明确的作用：

> cmake -G Xcode \ # 使用 Xcode 生成器
 -DLLVM\_ENABLE\_PROJECTS="clang;lld" \ # 启用 clang 和 lld 项目
 -DLLVM\_TARGETS\_TO\_BUILD="AArch64" \ # 只构建 ARM64 架构支持
 -DCMAKE\_BUILD\_TYPE=Release \ # Release 构建类型
 -DLLVM\_BUILD\_EXAMPLES=ON \ # 构建示例代码
 -DLLVM\_INCLUDE\_TESTS=OFF \ # 不包含测试
 -DLLVM\_ENABLE\_ASSERTIONS=OFF \ # 禁用断言
 -DLLVM\_OPTIMIZED\_TABLEGEN=ON \ # 优化 TableGen 构建
 -DLLVM\_USE\_SPLIT\_DWARF=ON \ # 减少调试信息大小
 -DLLVM\_PARALLEL\_COMPILE\_JOBS=8 \ # 并行编译作业数
 -DLLVM\_PARALLEL\_LINK\_JOBS=4 \ # 并行链接作业数
 ../llvm # LLVM 源码路径


![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/f467cca5-4297-4c1b-8546-caa32a4051fb.png)

需要注意的是，M1 电脑如果需要构建 x86\_64 架构，需要安装 x86\_64 架构的 libzstd。

### 配置 LLVM Xcode 工程

Xcode 编译工程生成后，打开 LLVM.xcodeproj 工程，会弹出下面弹框：
   

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/f2ae4d4e-e6c1-42aa-aa3c-36694a05d2d7.png)![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/91da8eb4-26ae-4237-86e8-b35b2cd940da.png)

此时应选择 “Manually Manage Schemes” 手动管理方案，因为默认的 schemes 数量过多，选择自动管理会导致工程卡顿，之后关闭 schemes 窗口，等待创建 Target 后再添加 scheme。

接着，添加一个 Aggregate 类型的 target 并命名为 InstrPass。    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/b43351eb-ee4f-4b66-ad31-ccd0e14954a1.png)

添加完成后，给 InstrPass 添加 opt 和 llc 依赖。    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/5c311559-eec3-4ad9-a131-183f723bb440.png)

最后，将 InstrPass target 添加到 schemes 中。
    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/c2ef8aa8-d12f-4c82-9b0c-2f06fb7ef4bf.png)

## Pass 编译流程

### 创建 LLVM Pass

创建自定义的 LLVM Pass 是实现插桩的核心步骤，需要按照规范将相关文件添加到 LLVM 源码目录并修改相应配置文件（Pass 编写参考官方文档 ：[https://llvm.org/docs/WritingAnLLVMNewPMPass.html](https://llvm.org/docs/WritingAnLLVMNewPMPass.html)）。


首先，将 `InstrPass.cpp` 复制到 `llvm-project-swift-release-6.0.2/llvm/lib/Transforms/Utils/` 目录下。

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/06032647-8517-4e9d-a194-e749d739ed78.png)

将 `InstrPass.h` 复制到 `llvm-project-swift-release-6.0.2/llvm/include/llvm/Transforms/Utils/` 目录下。

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/03a57832-9761-4e89-80e0-dfa560d119c3.png)

然后，修改 `llvm-project-swift-release-6.0.2/llvm/lib/Transforms/Utils/CMakeLists.txt` 文件，在其中添加 `InstrPass.cpp`，确保 CMake 在构建时能包含该文件。    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/e228cf28-376d-4825-a362-92d0408dc4e0.png)

之后，修改 `llvm-project-swift-release-6.0.2/llvm/lib/Passes/PassRegistry.def` 文件，添加 `FUNCTION_PASS("instr-pass", InstrPass())`，可以搜索 **HelloWorldPass** 并在其下方添加，这样可以将自定义的 Pass 注册到 LLVM 的 Pass 系统中。    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/89de68e2-6b5d-464d-89d1-873a16ccfe8f.png)

最后，修改 `llvm-project-swift-release-6.0.2/llvm/lib/Passes/PassBuilder.cpp` 文件，导入头文件：`#include "llvm/Transforms/Utils/InstrPass.h"`

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/970215fb-8416-4000-9aa9-e6f7c6d17c2d.png)

### 编译 opt 和 llc

回到 LLVM Xcode 工程，进行编译操作（按下 cmd + b）。构建成功后，在工程的 Debug/bin 目录下就能看到生成的 opt 和 llc 可执行文件（test 是我自己构建的[《test.ir 构建》](https://alidocs.dingtalk.com/i/nodes/qnYMoO1rWx7QxLDRibnzXAgb847Z3je9?utm_scene=person_space)）。

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/752f0eaf-ac51-42a8-81ea-04d319bac9ca.png)

### 验证 Pass 有效性

为了验证自己编写的 Pass 是否生效，可以使用 opt 命令处理 ir 文件。在终端中进入 Debug/bin 目录，执行以下命令：

```plaintext
cd Debug
cd bin
./opt test.ir -passes=instr-pass -o test2.ir
```

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/d43706d7-bbb8-4fe6-a867-308b901244f7.png)

通过查看执行命令后的输出结果或对比处理前后的 ir 文件，能够判断 Pass 是否正常工作。


## 项目插桩实践

### 安装插桩工具链

[请至钉钉文档查看附件《InstrPass.zip》](https://docs.dingtalk.com/i/nodes/Gl6Pm2Db8DAbj03gIqxMMpokWxLq0Ee4?iframeQuery=anchorId%3DX02megwv6dmit2nty0330b)

首先，下载插桩工具链，工具链位置在 `InstrPass/core/InstrPass/compile-script`。然后，打开终端并进入 compile-script 目录，执行 `sh ``install.sh` 脚本（终端需要开启磁盘访问权限）。脚本执行成功则代表工具链已经安装完成，若需要卸载工具链，执行 `sh ``uninstall.sh` 即可。

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/a83fe2c5-24e7-4266-bbb9-d33b22df773f.png)

### 添加配置文件并编译项目

在需要插桩的工程根目录添加 `instrpass.config` 文件。

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/8e3bbdf5-7c6a-4430-9585-31ff4619ec94.png)

文件内容如下：

```shell
#!/bin/bash
# This is a shell script configuration file for InstrPass

INSTPASS_ENABLED=1 # 1代表开启插桩，0表示不开启插桩

# 插桩目标文件列表 - 指定需要进行插桩处理的文件
INSTPASS_TARGETS=("ViewController")

# 插桩排除文件列表 - 指定不需要进行插桩处理的文件
INSTPASS_EXCLUDES=() #插桩排除文件列表

#指定自己编译的opt 和 llc
LLVM_PASSES=("instr-pass")
#LLVM_PASSES=("my-custom-pass" "my-custom-pass2") # 可以配置多个pass
LLVM_LLC_PATH="/Users/cityu/LLVM/llvm-project-swift-release-6.0.2/build/Debug/bin/llc"
LLVM_OPT_PATH="/Users/cityu/LLVM/llvm-project-swift-release-6.0.2/build/Debug/bin/opt"
```

在配置脚本中，可以根据需求指定是否开启插桩、插桩的目标文件列表、排除文件列表，以及自己编译的 llc 和 opt 路径、Pass 名称等。

接着，打开工程并添加 hook 函数。    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/31a758da-134c-42c4-ac0d-d43d18e0af1a.png)

完成后编译工程，查看 build 日志，找到 “Link” 阶段并打开。    

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/3dbd03ce-0c22-4546-964e-f622fedfd302.png)

找到如下图中内容，说明插桩成功，会打印出插桩函数的符号、参数及返回值类型，插桩失败通常会报错。

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Lk3lbmb08NVYMOm9/img/4bb27ca0-3304-4d62-827c-e8c5fda310bc.png)
