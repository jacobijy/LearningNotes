
在网络受限或需要穿越跳板机的场景下，可以通过 SSH 配置代理来实现远程访问。以下介绍两种常用且高效的方式。

方法一：使用 ProxyCommand / ProxyJump

**步骤 1：编辑 SSH 配置文件**

vim ~/.ssh/config

**步骤 2：添加代理配置**

- **ProxyJump 示例（推荐，高版本 OpenSSH 支持）**
    
`Host target-server`
	`HostName 203.0.113.10`
	`User root`
	`ProxyJump user@192.168.1.100`

- **ProxyCommand + nc 示例（兼容低版本 OpenSSH）**
    
`Host target-server`
	`HostName 203.0.113.10`
	`User root`
	`ProxyCommand nc -X 5 -x 127.0.0.1:1086 %h %p`
其中 _-X 5_ 表示 SOCKS5 协议，_-x_ 指定代理地址与端口。

**步骤 3：直接连接目标主机**

ssh target-server