# tunasync workers.conf 配置详解

以下配置推荐对照 worker 配置文件示例 [workers.conf](workers.conf) 查看。

带 * 号的配置是数组，需要使用["xxx","xxx"]格式配置

建议在使用 cgroup、docker、zfs、snapshot 等选项前先熟悉相关知识。

- `[global]`
  这是一个全局设置，对所有配置均生效
  - `mirror_dir`: 设置全局的同步路径，各个同步项目会尝试在该目录下创建同名子文件夹并同步
  - `concurrent`: 最大并发数量，即同时最多几个仓库同步（手动触发的不计）
  - `interval`: 间隔时间，即同步完成后下次同步的时间，单位是分钟
  - `retry`: 重试次数，在单次同步失败后最多尝试几次(无间隔时间)
  - `timeout`: 超时时间，即单次同步总时长到达该时间后强制失败退出，单位是秒
  - `log_dir`: 日志目录，可以用`{{.Name}}`设置目录名，使每个同步仓库的日志放在不同文件夹
  - `rsync_options`*: 全局增加的同步参数（仅对于rsync同步模式生效）
  - `exec_on_success`*: 同步成功后执行自定义的脚本
  - `exec_on_failure`*: 同步失败后执行自定义的脚本（注意，若有重试次数，则每次失败尝试均会执行）
  - `dangerous_global_success_exit_codes`*: 允许一些非致命的同步失败·视为成功（如 rsync 返回 23/24 等常见的“同步完成但有部分文件问题”）
  - `dangerous_global_rsync_success_exit_codes`*: 作用同上，但仅针对`rsync`同步方式有效
- `[manager]`
  - `api_base`: manager的访问端口，在`manager.conf`中配置
  - `api_base_list`*: 如果有多个manager，在此处以数组方式填写，所有信息均会同步发送到各个manager，填写该选项将覆盖`api_base`
  - `ca_cert`: 同上，证书设置（注意是公钥）
- `[zfs]`
  这是用来检查同步目录是否是通过 zfs 挂载了的。如果启用，那么会强制检查文件夹是否挂载到 zpool 上（使用`mountpoint /path`检查）。检查不通过的会直接同步失败，同时根据 zpool 参数输出建议创建的zpool名称，但***不会自动创建对应的 zpool ***
  - `enable`: 是否启用
  - `zpool`: 建议挂载的 zpool 前缀名
- `[cgroup]`
    请查看 [cgroup.md](../cgroup.md)
- `[btrfs_snapshot]`
  请查看 [tips.md](tips.md)
- `[docker]`
  此处为全局生效的 docker 配置选项
  - `enable`: 是否开启 docker
  - `volumes`: 挂载的卷
  - `Options`: 默认选项
- `[include]`: 包含各个仓库的 conf 文件，组成一个完整的配置
  - `include_mirrors`*: 需要添加的文件路径，可以使用`*`匹配，但是不能使用其他匹配，如`**`，示例：`"/tmp/tunasync123/*.conf"`
- `[[mirrors]]`
  各个仓库的具体设置，部分同名配置可省略使用全局配置。优先级: `mirrors` > `global`
  - `name`: 仓库名
  - `provider`: 设置同步方式，以下分别介绍各自的配置
    -  `provider = "command"`: 使用命令同步
       -  `command`: 同步脚本/命令
          以下是一些运行脚本添加的环境变量参数
          -  `TUNASYNC_MIRROR_NAME`: `name`
          -  `TUNASYNC_UPSTREAM_URL`: `upstream`
          -  `TUNASYNC_WORKING_DIR`: `mirror_dir`、`mirror_subdir`: 组合生成，同时这是命令的工作目录（目录生成方式见下文）
       -  `log_dir`: 输出日志的目录，未设置则使用全局设置
       -  `size_pattern`: 正则匹配仓库体积，需要脚本输出体积大小的日志
       -  `fail_on_match`: 正则匹配错误信息，部分脚本同步失败时也是退出码为0，使用该参数匹配错误信息，匹配成功即为同步失败
    -  `provider = "rsync"`: 使用 rsync
        工作目录见下方`mirror_dir`、`mirror_subdir`介绍
       -  `rsync_no_timeout`: 布尔值，是否不限制 rsync 同步时间，一般不推荐开启
       -  `rsync_timeout`: 设置 rsync 最长的超时时间（ rsync 参数的超时配置，非总时长）
       -  `rsync_options`*: 额外添加的其他参数
       -  `exclude_file`: 添加外部文本文件的列表排除同步目录，实际上仍然是拼接到`rsync --exclude`参数，也可以直接追加在`rsync_options`，也建议追加一个`rsync_options = [ "--delete-excluded" ]`删除被排除的文件
       -  `rsync_override`*: 直接覆盖tunasync的默认追加参数（默认参数见[源码](https://github.com/tuna/tunasync/blob/master/worker/rsync_provider.go)）
       -  `rsync_override_only`: 布尔值，是否只有`rsync_override`的参数。该参数**不是`rsync_override`的启用开关**。未启用时，会按照配置添加`rsync_timeout`、`rsync_options`等参数；启用时，不添加额外参数，仅有`rsync_override`的参数。**请确定已知晓自己在做什么！**
    -  `two-stage-rsync`:
    主要解决debian等大型项目，在同步时存在元数据文件与实际文件更新不一致的问题。分成两个阶段，先更新元数据，再更新其他文件
    -  `stage1_profile`: 设置同步方式，仅可设置为`"debian"`、`"debian-oldstyle"`
  -  `env`: 额外的环境变量，示例:
    ```ini
    # inline模式
    [[mirrors]]
    ...
    env = { REPO = "/path", DEBUG = "1" }
    # expanded模式
    [[mirrors]]
    ...
    [mirrors.env]
    REPO = "/path"
    DEBUG = "1"
    ```
  - `role`: 对于同步无意义，仅在生成json时控制布尔值`is_master`，仅能填写`master`或`slave`
  - `mirror_dir`: 同步的目录
  - `mirror_subdir`: 同步的子目录
    以下列出`mirror_dir`和`mirror_subdir`生成的工作目录情况
    |[global]mirror_dir|[mirrors]mirror_dir|[mirrors]mirror_subdir|工作目录|
    |---|---|---|---|
    |`/global`|||`/global/{{.Name}}`|
    |`/global`|`/mirror`||`/mirror`|
    |`/global`||`/mirror_sub`|`/global/mirror_sub/{{.Name}}`|
    |`/global`|`/mirror`|`/mirror_sub`|`/mirror`|
  - `interval`: 见`global`设置
  - `retry`: 见`global`设置
  - `timeout`: 见`global`设置
  - `exec_on_success`*: 见`global`设置
  - `exec_on_failure`*: 见`global`设置
  - `success_exit_codes`*: 见`global`设置，如果是 rsync 还会添加 rsync 的成功码，但建议 rsync 使用专用的`rsync_success_exit_codes`
  以上设置为覆盖性设置，可省略使用全局配置。优先级: `mirrors` > `global`
  - `exec_on_success_extra`*: 同`exec_on_success`，但是在`[global]exec_on_success`基础上增加。建议`mirror`内`exec_on_success_extra`和`exec_on_success`二选一设置
  - `exec_on_failure_extra`*: 同上
  - `use_ipv6`: 是否使用 ipv6
  - `use_ipv4`: 是否使用 ipv4，二者不可同时开启或关闭
  - `docker_image`: docker 镜像名称
  - `docker_volumes`*: 在`global`的基础上，追加 docker 目录映射
  - `docker_options`*: 在`global`的基础上，追加 docker 选项
  - `memory_limit`: 限制的内存大小，其实现依赖 docker 和 cgroup，未配置 docker 和 cgroup 则不生效。
  - `snapshot_path`: 快照目录