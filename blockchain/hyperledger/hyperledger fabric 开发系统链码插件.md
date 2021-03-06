# `hyperledger fabric` 系统链码插件

**系统链码是专用链码，作为对等进程的一部分运行，而不是在单独的`docker`容器中运行的用户链代码**。因此，他们可以更多地访问对等体中的资源，并且可以用于**实现难以或不可能通过用户链代码实现的功能特性**。系统链接的示例包括用于分类帐和其他与`Fabric`相关的查询的`QSCC`（查询系统链接），用于帮助调节访问控制的`CSCC`（配置系统链接）和`LSCC`（生命周期系统链接）。

与用户链代码不同，系统链代码**未使用`SDK`或`CLI`中的提议进行安装和实例化**。它在**初创时由对等方注册和部署**。

系统链代码可以通过两种方式链接到对等方：**静态和动态使用`Go`插件**。本教程将概述如何**开发和加载**系统链代码作为插件。

# 开发插件

**系统链代码是用`Go`编写的程序，使用[`Go`插件包](https://golang.org/pkg/plugin)加载**。插件包含一个带有导出符号的主包，并使用命令`go build -buildmode = plugin`构建。

每个系统链代码都**必须实现[`Chaincode`接口](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#Chaincode)**并输出一个与主包中的签名`func New() shim.Chaincode`匹配的**构造函数方法**。可以在`examples/plugin/scc`的存储库中找到一个示例。

诸如`QSCC`的现有链码还可以用作通常**通过系统链代码实现的某些特征的模板**，例如访问控制。现有的系统链代码也可作为**日志记录和测试**等最佳实践的参考。

> **注意**：在导入的包上：`Go`标准库要求插件必须包含与主机应用程序**相同的导入包版本**（在本例中为`Fabric`）。

# 配置插件

插件在`core.yaml`的`chaincode.systemPlugin`部分中配置：

```yaml
chaincode:
  systemPlugins:
    - enabled: true
      name: mysyscc
      path: /opt/lib/syscc.so
      invokableExternal: true
      invokableCC2CC: true
```

系统链码也必须在`core.yaml`的`chaincode.system`部分**列入白名单**：

```yml
chaincode:
  system:
    mysyscc: enable
```

