# 在 ECS 实例上部署 OpenClaw：三种部署方式怎么选，云助手命令参数怎么填、Windows 实例有哪些坑

OpenClaw 是一个开源的 AI 助手框架，支持接入多个模型，并把消息分发到不同的渠道。手动部署它通常要自己装依赖、配模型；在阿里云的 ECS 实例上部署 OpenClaw，这两步都有官方工具可以代劳。目前官方支持三条路：计算巢一键部署、云助手公共命令、源码编译。前两条覆盖绝大多数场景，源码编译只在你要改代码时才用得上。

> **快速结论**：第一次部署，直接走计算巢，填一张可视化表单，ECS 环境和 OpenClaw 会自动装好；要批量装、或者不想登录实例，用云助手公共命令。配置建议 4 vCPU / 4 GiB 内存起步。实例还没开的话，[👉 查看阿里云 ECS 当前的特惠机型](https://bit.ly/aliyun-coupon)，选好机器再回来跑命令。

## 部署前先核对这个清单

任何一条安装路径，都建立在这几个前提上，缺一个就可能中途卡住：

- **云助手 Agent 正常**：实例上已安装云助手 Agent，状态为“正常”。没装的话可以在 ECS 控制台补装。
- **能访问公网**：安装过程要拉取 OpenClaw 依赖包和模型配置，实例必须能出公网。
- **系统版本在支持列表里**：

| 操作系统 | 支持版本 | 公共命令 |
| --- | --- | --- |
| Linux | Alibaba Cloud Linux 3、Ubuntu 22.04/24.04、Debian 12、Rocky Linux 8/9、AlmaLinux 8/9、CentOS Stream 8/9 | `ACS-ECS-InstallOpenClaw-for-linux.sh` |
| Windows | Windows Server 2016 及之后版本 | `ACS-ECS-InstallOpenClaw-for-windows.ps1` |

规格方面，官方建议至少 4 vCPU 和 4 GiB 内存。共享型实例（ecs.t6、ecs.s6 这类）性能上限偏低，如果非用不可，往更大规格选，不然安装和运行都可能出现不稳定。

Windows 实例还有两条额外限制，本文后面单独讲，跑命令前建议先看一眼。

## 三种部署方式怎么选

结论先给：日常部署在计算巢和云助手命令里二选一，源码编译留给要二次开发的人。三种方式的差别如下：

| 部署方式 | 适合场景 | 优势 | 入口 |
| --- | --- | --- | --- |
| 计算巢 | 想一次开通直接用，支持试用 | 填可视化表单，ECS 与 OpenClaw 的安装配置全自动完成，方便接消息渠道 | [ 打开一键部署页](https://www.aliyun.com/solution/tech-solution-deploy/3025287?userCode=5eqf8rny) |
| 云助手公共命令 | 批量部署、不登录实例做运维 | 填参数即可完成安装、模型切换、渠道接入、健康诊断全流程，支持批量执行 | [ 开一台 ECS 实例再跑命令](https://bit.ly/aliyun-coupon) |
| 源码编译 | 深度定制、社区开发 | 可以改源码、调试功能，适合开发者和二次开发场景 | [ 开一台 ECS 实例自行编译](https://bit.ly/aliyun-coupon) |

简单说：只关心能不能尽快跑起来的选计算巢；管一批机器的选云助手命令；两边都不沾的才碰源码。

## 方式一：计算巢一键部署

计算巢把整条部署链路自动化了：不需要手动装依赖，也不需要配置实例环境，填一张可视化表单，ECS 和 OpenClaw 的安装配置自动完成，后续接入消息渠道也省事，还支持先试用。

操作路径很直接：打开部署方案页，按表单填写配置，提交后等它自动装完。完整步骤在方案页里有逐步说明，照着填就行。

[👉 打开计算巢的 OpenClaw 一键部署页面](https://www.aliyun.com/solution/tech-solution-deploy/3025287?userCode=5eqf8rny)

## 方式二：云助手公共命令，不登录实例也能装

这种方式的最大好处是不用 SSH 登录：在控制台或命令行填几个参数，安装、模型切换、渠道接入、健康诊断都能做，还支持一次对多台实例批量执行。

### 控制台怎么操作

1. 登录 ECS 控制台，进入云助手页面，顶部先切到目标实例所在的地域。
2. 切换到“公共命令”页签，搜索 `ACS-ECS-InstallOpenClaw`，找到对应命令卡片，点击执行。
3. 在执行窗口配置参数并勾选目标实例。三个关键参数：
   - `INSTALL_METHOD`：固定填 `plugin`。
   - `AUTH_CHOICE`：模型服务商标识，例如 `modelstudio-standard-api-key-cn` 对应通义千问标准版，完整列表见官方文档。
   - `API_KEY`：对应服务商的 API 密钥。官方建议改用 `ENCRYPTED_API_KEY` 加 `KEY_PAIR_ID` 的组合做加密传输，避免明文密钥直接出现在参数里。
4. 到“命令执行结果”页签看状态。执行成功即安装完成；失败就按任务输出和云助手日志排查。
5. 可选：安装时选了哪个服务商，装完后 OpenClaw 默认就用它的默认模型，之后可以再查看或切换。

### 命令行怎么操作

先查命令详情：


aliyun ecs DescribeCommands \
  --RegionId 'cn-hangzhou' \
  --CommandId 'ACS-ECS-InstallOpenClaw-for-linux.sh'


Windows 实例把 `CommandId` 换成 `ACS-ECS-InstallOpenClaw-for-windows.ps1`。

执行安装。地域 ID 在控制台左上角能查到，实例 ID 在实例列表页：


aliyun ecs InvokeCommand \
  --RegionId '<地域 ID，例如 cn-hangzhou>' \
  --CommandId 'ACS-ECS-InstallOpenClaw-for-linux.sh' \
  --InstanceId.1 '<实例 ID，例如 i-bp1xxxxxxxxxxxx>'


批量安装时用 `--InstanceId.N` 传多个实例 ID，上限 100 台。

命令执行后会返回一个 `InvokeId`，用它查结果：


aliyun ecs DescribeInvocationResults \
  --RegionId '<地域 ID>' \
  --InvokeId '<调用 ID，例如 t-hz0xxxxxxxxxxxx>'


返回里 `Output` 字段是脚本的标准输出，`ErrorInfo` 字段是错误信息，排查失败先看这两个。

## 装完之后：模型、渠道、网关都有对应的公共命令

安装只是第一步。后续的模型管理、消息渠道、网关服务、诊断和健康检查、卸载清理，官方都做成了同系列公共命令，同样不用进实例敲命令。

最常用的是模型查看和切换，命令是 `ACS-ECS-OpenClawModel-for-linux.sh`（Windows 对应 `windows.ps1`），通过 `ACTION` 参数控制：

- `ACTION=status`：查看当前默认模型和配置
- `ACTION=list`：列出已配置的模型
- `ACTION=switch`：切换默认模型

消息渠道管理、网关服务管理这些操作，同样在云助手的公共命令里搜到对应命令、按参数说明执行即可。

安装成功后如果想远程访问 OpenClaw 的 Dashboard（WebUI），需要按官方文档做公网访问配置。这里提前提醒一句：社区部署指南提到，网关绑定到非本地回环地址时必须配置网关令牌认证——也就是说，远程访问前先把认证配好，别把没有保护的服务直接暴露在公网上。

## Windows 实例：跑命令前先看这两条

Windows 不是装不了，是有限制，不提前处理，命令跑了也白跑。

1. **网关要等用户登录桌面后才启动**：OpenClaw 的 Gateway 在有用户登录桌面之后才会起来，所以执行安装或相关命令之前，先确认已有用户登录过桌面。
2. **不能用默认的 system 用户执行**：运行 Windows 命令时必须指定执行用户，比如 Administrator。想用普通用户跑，需要先按官方文档给 Windows 实例配置普通用户执行云助手命令的权限。

## 为什么 OpenClaw 需要一台常驻的服务器

这也是很多人纠结“要不要用 Serverless”的答案：OpenClaw 的 Gateway 是长驻进程，凭据、会话、配置这类状态都持久化保存在服务器本地。按请求运行的无服务器平台留不住这种状态，所以社区指南明确不建议用 Lambda 这类方案跑网关。

落到 ECS 上有两个实际推论：一是选好实例后尽量稳定用，重置系统盘这类操作会让本地状态全部清零、从头再来；二是以后如果迁移，记得把状态目录一起搬走，只装程序不搬数据是不够的。

## 常见问题

### 部署 OpenClaw 的 ECS 最低配置是多少？

官方建议至少 4 vCPU 和 4 GiB 内存。共享型实例（ecs.t6、ecs.s6 系列）建议选更大规格，保证安装和运行稳定。实例还没开的话，[👉 去特惠页看看当前的机型和价格](https://bit.ly/aliyun-coupon)。

### 安装失败怎么排查？

先看“命令执行结果”页签里的任务输出，再查云助手日志；CLI 场景用 `DescribeInvocationResults` 看 `ErrorInfo` 字段。网络是常见原因——实例必须能访问公网，否则拉不到依赖包和模型配置。

### 一条命令能同时装多台实例吗？

能。`InvokeCommand` 用 `--InstanceId.N` 参数传多个实例 ID，一次最多 100 台。

### 装完之后默认用什么模型？

安装时 `AUTH_CHOICE` 选的是哪家服务商，装完就默认用那家的默认模型。想换，用模型管理命令的 `ACTION=switch`。

### AWS 上能不能部署 OpenClaw？

可以，社区有 EC2 和容器部署的完整指南：EC2 上用 Ubuntu 22.04 加 Node.js 22，npm 全局安装后跑初始化向导；容器化部署要给状态目录挂持久化存储。但按请求计费的 Lambda 不适合跑 Gateway，原因见上文。本文只覆盖阿里云 ECS 的流程。

## 按你的情况对号入座

- 只想尽快用起来：走计算巢，填表单等结果，全程不碰命令行。
- 手里有一批实例，或者习惯脚本化运维：用云助手公共命令，API 密钥记得走加密传输。
- 要改源码做二次开发：直接源码编译，前面两种方式对你没有约束。
- 实例还没买：配置别低于 4 vCPU / 4 GiB，共享型（t6/s6）要么避开、要么往大规格选；Windows 实例记得桌面登录和执行用户这两个前提。[👉 去阿里云特惠页选一台合适的 ECS](https://bit.ly/aliyun-coupon)，开机回来照着上面的步骤跑就行。
