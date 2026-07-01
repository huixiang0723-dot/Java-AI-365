# Linux Day01：磁盘空间占满导致 Java 服务启动失败排障

日期：2026-07-01

## 一、问题现象

启动 Java 服务时执行：

```bash
./start.sh
```

出现错误：

```bash
No space left on device
```

同时脚本提示无法写入：

```bash
/core_java/rto_project_daqing_12021/app.pid
```

这说明服务启动过程中需要写入 PID 文件或日志文件，但服务器磁盘空间不足，导致写入失败。

------

## 二、第一步：确认是不是磁盘满

使用命令：

```bash
df -h
```

作用：查看服务器各个磁盘分区的空间使用情况。

排查结果：

```bash
/dev/sda3  116G  116G  63M  100% /
```

说明根分区 `/` 已经 100% 占满，只剩 63M 可用空间。

结论：

```text
不是 Java 代码问题，而是服务器根分区磁盘空间不足。
```

------

## 三、第二步：排除 inode 是否耗尽

使用命令：

```bash
df -i
```

作用：查看 inode 使用情况。

排查结果：

```bash
/dev/sda3  918912  473073  445839  52% /
```

说明 inode 只用了 52%，不是 inode 耗尽。

结论：

```text
问题不是小文件数量太多，而是大文件或大目录占满了磁盘空间。
```

------

## 四、第三步：定位哪个目录占空间

从根目录开始排查：

```bash
du -xh --max-depth=1 / 2>/dev/null | sort -hr
```

作用：

- `du`：统计目录大小。
- `-x`：只统计当前文件系统，避免跨挂载点。
- `-h`：以 GB、MB 的方式显示。
- `--max-depth=1`：只看当前目录下一层。
- `2>/dev/null`：忽略无权限等错误输出。
- `sort -hr`：按大小倒序排列。

排查结果：

```bash
115G /
88G  /var
13G  /opt
```

说明 `/var` 是最大目录。

继续排查：

```bash
du -xh --max-depth=1 /var 2>/dev/null | sort -hr
```

结果：

```bash
88G /var
57G /var/opt
29G /var/lib
```

说明主要占用来自：

```text
/var/opt
/var/lib
```

继续排查 GitLab：

```bash
du -xh --max-depth=1 /var/opt/gitlab 2>/dev/null | sort -hr
```

结果：

```bash
57G /var/opt/gitlab
54G /var/opt/gitlab/backups
```

继续排查 Docker：

```bash
du -xh --max-depth=1 /var/lib 2>/dev/null | sort -hr
```

结果：

```bash
29G /var/lib
26G /var/lib/docker
3.3G /var/lib/mysql
```

最终定位：

```text
第一大头：/var/opt/gitlab/backups，占 54G
第二大头：/var/lib/docker，占 26G
```

------

## 五、根因分析

这次磁盘满的根因是：

```text
GitLab 每天自动生成备份文件，但没有及时清理旧备份。
```

备份目录：

```bash
/var/opt/gitlab/backups
```

其中存在大量备份文件：

```bash
*_gitlab_backup.tar
```

每个近期备份大约 900M，每天生成一个。

如果每天生成 900M：

```text
900M × 30 天 ≈ 27G
900M × 60 天 ≈ 54G
```

这与实际看到的 54G 完全吻合。

------

## 六、GitLab 是什么

GitLab 可以理解为公司内部自建的 GitHub。

它通常用于：

- 存放公司源代码。
- 管理开发人员权限。
- 管理代码分支。
- 发起 Merge Request。
- 执行 CI/CD。
- 备份 Git 仓库和相关数据。

目录：

```bash
/var/opt/gitlab
```

是 GitLab 的运行数据目录。

其中：

```bash
/var/opt/gitlab/backups
```

是 GitLab 的备份目录。

这个目录里的备份文件一般不是正在运行的数据，但它们可能是公司灾备策略的一部分，所以不能擅自删除。

------

## 七、为什么不能直接删除

虽然 `/var/opt/gitlab/backups` 是备份目录，但是否可以删除取决于公司备份策略。

例如公司可能要求：

```text
保留 7 天
保留 30 天
保留 90 天
同步到 NAS 后本地只保留 7 天
```

如果没有确认策略就直接删除，可能会导致：

- 违反公司备份要求。
- 出现事故时无法恢复 GitLab。
- 删除了仍然需要保留的历史备份。

生产环境原则：

```text
定位问题可以自己做。
涉及删除业务数据或备份数据，必须先确认策略。
```

------

## 八、安全查看可删除文件

可以先执行：

```bash
find /var/opt/gitlab/backups -type f -mtime +7 -name "*.tar" -print
```

这个命令是安全的，因为最后是：

```bash
-print
```

它只会打印符合条件的文件，不会删除。

含义：

- `find`：查找文件。
- `/var/opt/gitlab/backups`：查找目录。
- `-type f`：只查普通文件。
- `-mtime +7`：修改时间超过 7 天。
- `-name "*.tar"`：文件名以 `.tar` 结尾。
- `-print`：打印出来，不删除。

这个命令适合用来预览：

```text
如果我要删除 7 天以前的备份，会涉及哪些文件？
```

------

## 九、真正删除前的正确流程

如果公司确认只保留最近 7 天备份，可以执行：

```bash
find /var/opt/gitlab/backups -type f -mtime +7 -name "*.tar" -print
```

确认输出无误后，再执行：

```bash
find /var/opt/gitlab/backups -type f -mtime +7 -name "*.tar" -delete
```

但生产环境中建议先和负责人确认：

```text
GitLab backup 保留策略是多少？
是否已经同步到 NAS 或其他备份服务器？
本地是否可以只保留最近 7 天或 30 天？
```

------

## 十、Docker 清理命令

查看 Docker 空间占用：

```bash
docker system df
```

相对安全的清理命令：

```bash
docker system prune -f
```

它会清理：

- 停止的容器。
- 无用网络。
- 悬空镜像。
- 构建缓存。

它不会删除：

- 正在运行的容器。
- 默认不会删除 volume。

不要随便执行：

```bash
docker system prune -a
```

因为它会删除更多未使用镜像，生产环境风险更高。

不要手动删除：

```bash
rm -rf /var/lib/docker/overlay2
```

这可能破坏 Docker 容器和镜像。

------

## 十一、系统日志清理命令

查看 systemd 日志占用：

```bash
journalctl --disk-usage
```

清理旧日志，保留 500M：

```bash
journalctl --vacuum-size=500M
```

这个命令一般比较安全，因为它只清理 systemd 旧日志，不会删除 Java 服务、GitLab、Docker 或数据库。

------

## 十二、服务是否真正启动成功

不能只看启动脚本输出：

```bash
Started. PID=xxx
```

因为脚本打印成功不代表 Spring Boot 已经完全启动。

需要继续验证：

```bash
ps -ef | grep rto_project_daqing.jar
```

作用：查看 Java 进程是否存在。

```bash
netstat -tunlp | grep 12021
```

作用：查看服务端口是否正在监听。

```bash
tail -100 /core_java/rto_project_daqing_12021/app.log
```

作用：查看 Spring Boot 启动日志。

如果看到类似：

```text
Started xxxApplication
Tomcat started on port(s): 12021
```

才说明应用基本启动成功。

------

## 十三、这次问题的完整链路

```text
Java 服务启动失败
        ↓
start.sh 报 No space left on device
        ↓
df -h 发现根分区 / 100%
        ↓
df -i 排除 inode 耗尽
        ↓
du 从 / 一层层定位
        ↓
发现 /var 占用最大
        ↓
发现 /var/opt/gitlab 占用 57G
        ↓
发现 /var/opt/gitlab/backups 占用 54G
        ↓
确认 GitLab 每天自动备份但没有清理旧备份
        ↓
磁盘被 GitLab 历史备份占满
```

------

## 十四、向上级反馈的话术

可以这样说：

```text
我这边排查了一下，应用启动失败的直接原因是根分区空间不足。
继续定位后发现，/var/opt/gitlab/backups 目录占用了约 54G，里面是 GitLab 每天自动生成的备份文件。

目前看起来是 GitLab 备份长期累积，没有自动清理旧备份。
我想确认一下公司对 GitLab 备份的保留策略，是保留 7 天、30 天还是更长时间？

确认策略后，我再按要求清理旧备份，避免误删仍然需要保留的数据。
```

如果领导说“你看着删”，可以继续确认：

```text
我建议本地只保留最近 7 天或 30 天备份，删除更早的历史备份。
请您确认一下保留周期，我再执行清理。
```

------

## 十五、我的学习总结

这次问题不是 Java 代码问题，而是一次典型的 Linux 生产环境磁盘排障问题。

核心收获：

1. `No space left on device` 优先查看磁盘空间。
2. `df -h` 用来判断哪个分区满了。
3. `df -i` 用来判断 inode 是否耗尽。
4. `du -xh --max-depth=1` 用来一层层定位大目录。
5. 服务启动成功不等于问题解决，还要看磁盘是否仍然危险。
6. 生产环境不能盲目删除文件，尤其是 GitLab、Docker、数据库目录。
7. 涉及备份清理时，要先确认公司保留策略。
8. 真正的解决方案不是临时删除，而是建立自动清理策略。

------

## 十六、常用命令速查

```bash
# 查看磁盘空间
df -h

# 查看 inode
df -i

# 查看根目录各目录大小
du -xh --max-depth=1 / 2>/dev/null | sort -hr

# 查看 /var 下各目录大小
du -xh --max-depth=1 /var 2>/dev/null | sort -hr

# 查看 GitLab 目录占用
du -xh --max-depth=1 /var/opt/gitlab 2>/dev/null | sort -hr

# 查看 Docker 占用
docker system df

# 安全预览 7 天以前的 GitLab 备份
find /var/opt/gitlab/backups -type f -mtime +7 -name "*.tar" -print

# 删除 7 天以前的 GitLab 备份，执行前必须确认公司策略
find /var/opt/gitlab/backups -type f -mtime +7 -name "*.tar" -delete

# 查看 systemd 日志占用
journalctl --disk-usage

# 清理 systemd 旧日志
journalctl --vacuum-size=500M

# 查看 Java 进程
ps -ef | grep rto_project_daqing.jar

# 查看端口监听
netstat -tunlp | grep 12021

# 查看启动日志
tail -100 /core_java/rto_project_daqing_12021/app.log
```