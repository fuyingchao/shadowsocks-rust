> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_45270849/article/details/139238430?csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22139238430%22%2C%22source%22%3A%22qq_45270849%22%7D)

前言
--

![](https://img-blog.csdnimg.cn/direct/2a2729e32539451cb5d5c2aedf4ff725.png)简称 SS，是一种基于 SOCKS5 代理方式的加密传输协议。SS 节点就是使用 SS 协议进行通信的服务器。

在日常研究工作中，可能面临需要挂 vpn 才能爬取的网站，此时需要先让自己的电脑能够访问网站，然后才能实现爬取，但是当使用的节点 IP 被爬取网站封禁时，需要手动更换节点重新开始爬取。为了更高效地实现爬取，避免节点被封禁后代码停止，可以通过维护一个节点代理服务器池，在一个节点被封禁时自动更换其他节点。

步骤
--

首先，假设你已经有若干代理服务器节点的信息，从中取出几个用于构建代理服务器池。需要对这些节点信息进行解析。

```
class Proxy(BaseModel):
    name: str
    type: str
    server: str
    port: int
    cipher: str
    password: str
    udp: bool
 
class Config(BaseModel):
    proxies: List[Proxy]
 
config_str = """
proxies:
  - {"name":"...","type":"...","server":"...","port":...,"cipher":"...","password":"...","udp":...}
  - {"name":"...","type":"...","server":"...","port":...,"cipher":"...","password":"...","udp":...}
  - {"name":"...","type":"...","server":"...","port":...,"cipher":"...","password":"...","udp":...}
"""#使用自己的节点信息
 
# 将字符串形式的配置信息加载为字典
config_dict = yaml.safe_load(config_str)
 
# 使用 pydantic 解析配置
try:
    config = Config(**config_dict)
except ValidationError as e:
    print(f"Validation error: {e}")
```

下载支持命令行的![](https://img-blog.csdnimg.cn/direct/2a2729e32539451cb5d5c2aedf4ff725.png) 客户端（例如 [ss-rust](https://github.com/shadowsocks/shadowsocks-rust/releases/tag/v1.19.0 "ss-rust")，该压缩包中没有可执行文件，可以在我的代码仓库中获取完整压缩包 [ss-rust](https://github.com/fuyingchao/shadowsocks-rust "ss-rust")），下载完整后将解压缩位置添加进系统的环境变量中。

接着就可以在 python 中启动![](https://img-blog.csdnimg.cn/direct/2a2729e32539451cb5d5c2aedf4ff725.png)，并随机选取节点信息用于配置![](https://img-blog.csdnimg.cn/direct/2a2729e32539451cb5d5c2aedf4ff725.png)。配置完成后，就可以使用该节点的代理服务器向目标网站发送请求了。以下是随机选取节点并配置![](https://img-blog.csdnimg.cn/direct/2a2729e32539451cb5d5c2aedf4ff725.png)的函数。

```
def get_random_proxy(proxies):
    """
    Randomly selects a proxy from the given list.
    Handles the case when a proxy is blocked by trying another one.
    """
    selected_proxy = random.choice(proxies)
    print(f"Selected Proxy: {selected_proxy.name} -> Server: {selected_proxy.server}, Port: {selected_proxy.port}")
    # 创建 Ss 客户端配置
    ss_config = {
        "servers": [
            {
                "address": selected_proxy.server,
                "port": selected_proxy.port,
                "password": selected_proxy.password,
                "method": selected_proxy.cipher
            }
        ],
        "local_address": "127.0.0.1",
        "local_port": 1080
    }
 
    # 获取当前Python文件的目录
    current_dir = os.getcwd()
    ss_config_path = os.path.join(current_dir, 'ss_config.json')
    # 将配置写入文件
    with open(ss_config_path, 'w') as f:
        json.dump(ss_config, f)
 
    # 启动 Ss 客户端
    ss_local_command = f"sslocal.exe -c {ss_config_path}"
    process = subprocess.Popen(ss_local_command, shell=True)
 
    # 等待 Ss 客户端启动
    time.sleep(3)
 
    # 设置 HTTP 代理为本地 Ss 客户端
    proxies = {
        "http": "socks5h://127.0.0.1:1080",
        "https": "socks5h://127.0.0.1:1080"
    }
    # 返回代理配置和进程
    return proxies, process
```

为了使用方便，将上述功能封装到一个 get 函数中，这样就可以在其他文件中直接调用此函数。完整的文件代码如下：

```
from pydantic import BaseModel, ValidationError
from typing import List
import yaml
import requests
import random
import subprocess
import time
import json
import os
import psutil
 
class Proxy(BaseModel):
    name: str
    type: str
    server: str
    port: int
    cipher: str
    password: str
    udp: bool
 
class Config(BaseModel):
    proxies: List[Proxy]
 
config_str = """
proxies:
  - {"name":"...","type":"...","server":"...","port":...,"cipher":"...","password":"...","udp":...}
  - {"name":"...","type":"...","server":"...","port":...,"cipher":"...","password":"...","udp":...}
  - {"name":"...","type":"...","server":"...","port":...,"cipher":"...","password":"...","udp":...}
"""#使用自己的节点信息
 
# 将字符串形式的配置信息加载为字典
config_dict = yaml.safe_load(config_str)
 
# 使用 pydantic 解析配置
try:
    config = Config(**config_dict)
except ValidationError as e:
    print(f"Validation error: {e}")
 
def find_and_kill_process_by_port(port):
    for proc in psutil.process_iter(['pid', 'name', 'connections']):
        try:
            for conn in proc.info['connections']:
                if conn.laddr.port == port:
                    print(f"Found process {proc.info['name']} (PID: {proc.info['pid']}) using port {port}")
                    os.kill(proc.info['pid'], 9)  # 强制终止进程
                    print(f"Process {proc.info['name']} (PID: {proc.info['pid']}) has been terminated")
                    return True
        except (psutil.NoSuchProcess, psutil.AccessDenied, psutil.ZombieProcess):
            continue
    print(f"No process found using port {port}")
    return False
 
# 结束使用端口1080的进程
find_and_kill_process_by_port(1080)
 
    
def get_random_proxy(proxies):
    """
    Randomly selects a proxy from the given list.
    Handles the case when a proxy is blocked by trying another one.
    """
    selected_proxy = random.choice(proxies)
    print(f"Selected Proxy: {selected_proxy.name} -> Server: {selected_proxy.server}, Port: {selected_proxy.port}")
    # 创建 Ss 客户端配置
    ss_config = {
        "servers": [
            {
                "address": selected_proxy.server,
                "port": selected_proxy.port,
                "password": selected_proxy.password,
                "method": selected_proxy.cipher
            }
        ],
        "local_address": "127.0.0.1",
        "local_port": 1080
    }
 
    # 获取当前Python文件的目录
    current_dir = os.getcwd()
    ss_config_path = os.path.join(current_dir, 'ss_config.json')
    # 将配置写入文件
    with open(ss_config_path, 'w') as f:
        json.dump(ss_config, f)
 
    # 启动 Ss 客户端
    ss_local_command = f"sslocal.exe -c {ss_config_path}"
    process = subprocess.Popen(ss_local_command, shell=True)
 
    # 等待 Ss 客户端启动
    time.sleep(3)
 
    # 设置 HTTP 代理为本地 Ss 客户端
    proxies = {
        "http": "socks5h://127.0.0.1:1080",
        "https": "socks5h://127.0.0.1:1080"
    }
    # 返回代理配置和进程
    return proxies, process
 
def get(url: str, headers: dict[str, str] = None, cookies: dict[str, str] = None, params: dict[str, str] = None):
    """
    Sends a request to the given URL using a randomly selected proxy.
    Retries with another proxy if the request fails.
    """
    session = requests.Session()
    while True:
        try:
            # 尝试发送请求
            proxies, process = get_random_proxy(config.proxies)
            session.proxies.update(proxies)
            response = session.get(url, headers=headers, cookies=cookies, params=params)
            print(response.text)
            process.kill()
            process.wait()
            return response
            break  # 如果成功获取响应，则退出循环
        except Exception as e:
            print(f"Error: {e}")
            print("Retrying with another proxy...")
            process.kill()
            process.wait()
            continue  # 如果出现异常，则继续尝试使用另一个代理
 
 
# 示例调用
if __name__ == "__main__":
    get('https://www.example.com')
```