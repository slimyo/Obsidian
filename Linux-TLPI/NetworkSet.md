

---

## **方案 1（推荐）：使用 Windows 局域网 IP 或 WSL2 网关 IP 配置代理**

1. 在 WSL2 里获取 Windows 虚拟网关 IP：
    

`ip route | grep default`

输出示例：

`default via 172.30.112.1 dev eth0`

- `172.30.112.1` 就是 Windows 主机 IP
    

2. 设置代理（假设代理端口 7890）：
    

`export http_proxy="http://172.30.112.1:7890" export https_proxy="http://172.30.112.1:7890"`

3. 测试代理是否生效：
    

`curl https://developer.nvidia.com -v`

如果能返回 HTML → 代理配置成功。

永久生效：

`echo 'export http_proxy="http://172.30.112.1:7890"' >> ~/.bashrc echo 'export https_proxy="http://172.30.112.1:7890"' >> ~/.bashrc source ~/.bashrc`