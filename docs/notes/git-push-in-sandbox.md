---
layout: page
title: 在受限环境里用 Git 推送 GitHub
---

*2026-10-02*

在一台被沙箱限制的 Windows 机器上，从零把代码推上 GitHub，并解决两个真实障碍。

## 起点

这台电脑上：

- 没有装 Git
- 命令行的 HTTPS 请求全部失败，`curl`、PowerShell 都报 `schannel: AcquireCredentialsHandle failed`
- 终端里 Git 弹出「请输入用户名」的提示**永远出不来**

## 坑一：Windows 原生 TLS 拿不到凭证

**现象**

```
fatal: unable to access 'https://github.com/...':
schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS
```

**原因**

Git for Windows 2.56 默认使用系统 TLS（schannel）。在受限进程里，它拿不到系统加密凭证，
于是所有 HTTPS 操作直接失败——但**明文 HTTP 是正常的**，这个对比是定位问题的关键线索。

**解决**

改用 Git 自带的 OpenSSL：

```bash
git config --global http.sslBackend openssl
```

## 坑二：交互式提示弹不出来

**现象**

```
fatal: 对路径"...\Temp\system-commandline-sentinel-files"的访问被拒绝。
fatal: could not read Username for 'https://github.com'
```

**原因**

Git 要弹出提示让你输用户名时，需要先在自己的临时目录写一个协调文件。
受限进程不能写工作区以外的路径，于是这个文件创建失败，提示根本弹不出来，命令直接报错退出。

**解决**

不要让 Git 去问，而是**提前把凭据交给它**：

```bash
git config --global credential.helper wincred
```

令牌成功使用过一次后，会被加密存进 Windows 凭据管理器（DPAPI 按当前用户加密），
之后 `git push` 不再询问任何东西。

如果连凭证助手都调不起来，还有一条兜底：把令牌交给一个隐藏输入脚本，
由脚本直接喂给 Git——令牌不打在屏幕上，也不进命令行历史。

## 三个可复用的经验

**1. 先排除自己的错误假设**

我一开始判断「本机代理端口死了、所以浏览器上不了网」，并准备去改系统代理。
后来发现本机的 `Get-NetTCPConnection` 在受限环境下**静默返回空**，
看起来像「端口没在监听」，实际是命令本身受限。

> 判断端口存活要用 `netstat -ano`，或者真的去连一次。**不要相信一个不报错但没输出的命令。**

**2. 区分「权限被改坏」和「进程被限制」**

遇到「访问被拒绝」，第一反应容易是「权限坏了，去修」。
但读了 ACL 才发现：`Temp`、`AppData`、用户目录的权限**完全正常**，
当前用户对它们都有完全控制——问题不是权限缺失，而是进程被限制在工作区内。

> 症状相同，病因完全不同。**先读证据，再动手修。**

**3. 交互式提示是自动化里最脆弱的一环**

凡是「停下来等人输入」的步骤，在受限或无人值守的环境里都容易崩。
更稳的做法是**预置凭据**或**隐藏输入**，让流程不需要任何交互。

## 一句话总结

HTTPS 失败先怀疑 TLS 后端，提示弹不出来先怀疑临时目录权限，
「拒绝访问」先分清是权限坏了还是进程被限制了。
