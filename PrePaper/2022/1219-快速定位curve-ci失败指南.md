# 快速定位 curve ci 失败指南

本文将主要介绍社区人员在参与curve项目开发过程中如何快速提交PR并进行CI测试，以及快速定位CI失败原因。

## 如何提交PR

在完成代码编写后，就可以提交 PR。当然如果开发尚未完成，在某些情况下也可以先提交 PR，比如希望先让社区看一下大致的解决方案，可以在完成代码框架后提价 PR。

[Curve 编译环境快速构建](https://github.com/opencurve/curve/blob/master/docs/cn/build_and_run.md)

[Curve 测试用例编译及执行](https://github.com/opencurve/curve/blob/master/docs/cn/build_and_run.md#测试用例编译及执行)

对于 PR 我们有如下要求：

- CURVE编码规范严格按照[Google C++开源项目编码指南](https://zh-google-styleguide.readthedocs.io/en/latest/google-cpp-styleguide/contents/)来进行代码编写，请您也遵循这一指南来提交您的代码。
- 代码必须有测试，文档除外，单元测试（增量行覆盖80%以上，增量分支覆盖70%以上）；集成测试（与单元测试合并统计，满足单元测试覆盖率要求即可）
- 请尽可能详细的填写 PR 的描述，关联相关问题的 issuse，PR commit message 能清晰看出解决的问题。
- CI 通过之后可开始进行 review，每个 PR 在合并之前都需要至少得到两个 Committer/Maintainer 的 LGTM。
- PR 代码需要一定量的注释来使代码容易理解，且所有注释和 review 意见和回复均要求使用英语。

对于 commit message：

一条好的 commit message 需要包含以下要素：

1. What is your change?（必须）
2. Why this change was made?（必须）
3. What effect does the commit have? (可选)

说明提交的 PR 做了那些修改：性能优化？修复bug？增加功能？以及这么做的原因。
最后描述以下这样修改带来的影响，包括性能等等。当然对一些简单的修改，修改的原因和影响可以忽略。
在 message中尽量遵循以下原则：

- 总结说明 PR 的功能和作用
- 使用短句和简单的动词
- 避免长的复合词和缩写

在提交时请尽可能的遵循以下格式：

```text
[type]<scope>: <description>
<BLANK LINE>
[body]
<BLANK LINE>
[footer]
```

type 可以是以下类型之一：

- build: 影响系统构建以及外部依赖
- ci: 影响持续继承相关的功能
- docs: 文档相关的修改
- feat: 增加新的特性
- fix: bug 修复
- perf: 性能提升
- refactor: 重构相关的代码，不增加功能也不修复错误
- style: 不影响代码的的含义的修改，仅仅是修改代码风格
- test: 单元测试相关的修改

第一行表示标题应尽可能保持 70 个字符以内，阐述修改的模块以及内容，多模块可以使用 `*` 来表示，并在正文阶段说明修改的模块。

footer 是可选的，用来记录对应因为这些更改而可以关闭的 issue，如 `Close #12345`

## CI 测试过程

PR提交到 Curve master 分支之后会自动触发Curve CI，需保证 CI 通过，CI 的 Jenkins 用户名密码为 netease/netease，如遇到 CI 运行失败可以登录 Jenkins 平台查看失败原因。测试内容包括：

### 1. cpplint 静态检查

主要进行测试内容为：

```bash
  cpplint --linelength=80 --counting=detailed --output=junit --filter=-build/c++11 --quiet $( find . -name *.h -or -name*.cc -or -name *.cpp ) 2>&1 | tee cpplint-style.xml
```

### 2. 单元测试

单元测试主要执行的就是仓库中：<https://github.com/opencurve/curve/blob/master/ut.sh>

为提升测试效率，所有单元测试用例为并行执行，需要注意增加的单元测试里起的服务不能和其他用例里起的服务使用同一端口，否则会因为端口占用失败

测试结束后进行覆盖率的卡点校验，当前覆盖率配置为：

```bash
if [ $1 == "curvebs" ];then

check_repo_branch_coverage 59
check_repo_line_coverage 76

## two arguments are module and expected branch coverage ratio
check_module_branch_coverage "src/mds" 70
check_module_branch_coverage "src/client" 78
check_module_branch_coverage "src/chunkserver" 65
check_module_branch_coverage "src/snapshotcloneserver" 65
check_module_branch_coverage "src/tools" 65
check_module_branch_coverage "src/common" 65
check_module_branch_coverage "src/fs" 65
check_module_branch_coverage "src/idgenerator" 79
check_module_branch_coverage "src/kvstorageclient" 70
check_module_branch_coverage "src/leader_election" 100
check_module_branch_coverage "nebd" 75

elif [ $1 == "curvefs" ];then

check_module_branch_coverage "mds" 59
check_module_branch_coverage "client" 59
check_module_branch_coverage "metaserver" 65
check_module_branch_coverage "common" 16
check_module_branch_coverage "tools" 0
fi
```

测试结束后会输出覆盖率报告，例如：[Report](http://59.111.91.248:8080/job/curve_untest_job/6149/HTML_20Report)。

如果测试失败，可以打开对应的curve_untest_job的控制台输出日志，直接页面拉到最后会打印出错误的原因，覆盖率不足会打印在最后，如果是单元测试用例失败，可以从下往上翻页，找到失败的地方，比如：

```text
[  FAILED  ] 1 test, listed below:
[  FAILED  ] EtcdClientTest.GetEtcdClusterStatus

 1 FAILED TEST
test bazel-bin/test/mds/server/mds-test.log log is --------------------------------------------->>>>>>>>
Running main() from gmock_main.cc
[==========] Running 1 test from 1 test case.
[----------] Global test environment set-up.
[----------] 1 test from MDSTest
[ RUN      ] MDSTest.common
2022-12-07 18:02:48.753204 I | etcd do NewCient get err:context deadline exceeded, errCode:DeadlineExceeded
2022-12-07 18:02:49.753500 I | etcd do NewCient get err:context deadline exceeded, errCode:DeadlineExceeded
2022-12-07 18:02:50.753831 I | etcd do NewCient get err:context deadline exceeded, errCode:DeadlineExceeded
2022-12-07 18:02:51.754211 I | etcd do NewCient get err:context deadline exceeded, errCode:DeadlineExceeded
2022-12-07 18:02:52.755814 I | etcd do NewCient get err:context deadline exceeded, errCode:DeadlineExceeded
2022-12-07 18:02:53.756543 I | etcd do NewCient get err:context deadline exceeded, errCode:DeadlineExceeded
test/mds/server/mds_test.cpp:76: Failure
Value of: initSuccess
  Actual: false
Expected: true
```

这里就表示：单元测试 MDSTest.common 运行失败，运行失败于文件 test/mds/server/mds_test.cpp 第 76 行，代码如下：

```c++
ASSERT_TRUE(initSuccess);
```

期望的值为： `true` ，实际的值为： `False` 。

### 3. pjdfstest 测试

pjdfstest 主要是用来测试 curvefs 在 POSIX上 的兼容性，会部署单机版的 curvefs 环境，然后进行 [pjdtest](https://github.com/pjd/pjdfstest) 的执行。
可以使用如下脚本进行自行测试：

```bash
#!/usr/bin/env bash

set -e

git clone https://github.com/pjd/pjdfstest.git
cd pjdfstest
autoreconf -ifs
./configure
make pjdfstest
cd ..
mkdir tmp
cd tmp
# must be root!
sudo prove -r -v --exec 'bash -x' ../pjd*/tests
cd ..
rm -rf tmp pjd*
```

如果测试失败，也需要进入控制台输出，拉到最后查看具体失败的原因。

### 4. failover 异常自动化

异常自动化会进行整体真实环境部署和io注入、故障测试，主要流程:

主要流程：

 初始化集群  →    用户io注入 （数据面） →   并发调用管理接口（管控面） →  触发异常  →  sleep a time  →  异常恢复(恢复校验)  → 数据校验（ioerror、读写一致性、三副本一致性、抖动） → 管理接口校验 &资源回收等校验

测试完成后会输出整体的测试报告，具体测试过程和测试结果可以点击对应提交的curve_failover_job 下的log.html , 可以看到比较完整的测试流程、步骤、操作和结果输出，比如 [log.html](http://59.111.91.248:8080/job/curve_failover_job/2579/robot/report/log.html)

## 常见问题

1. 如何登录jenkins?
由于内部的一些权限要求，需要用用户名密码登录 jenkins ，开源社区的可以使用 netease/netease 的账号/密码进行登录

2. 如果再次触发失败pr的CI 构建？
可以打开pr ，在最底部comment  `cicheck` 字段，进行再次触发构建

3. 如何过滤CI？
对于一些文档修改或其他不影响代码流程的内容，我们可以不进行CI构建，可以在创建PR的时候在标题增加 [skipci] 字段

4. 触发构建后立刻返回了失败？
查看一下失败的job内容，多数是因为测试任务在进行排队等待。如果job是 waiting_in_line ，就是在排队
