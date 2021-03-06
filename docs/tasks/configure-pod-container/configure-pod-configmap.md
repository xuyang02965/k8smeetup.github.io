---
标题: 在 Pod 里使用 ConfigMap 数据
---

{% capture overview %}
这篇教程提供了一系列的例子，展示如何配置 Pod 来使用 ConfigMaps 里面的数据。
{% endcapture %}

{% capture prerequisites %}
* {% include task-tutorial-prereqs.md %}
* [创建一个ConfigMap](/docs/tasks/configure-pod-container/configmap/)
{% endcapture %}

{% capture steps %}



## 使用 ConfigMap 数据来定义 Pod 环境变量

### 从单个 ConfigMap 提取数据定义 Pod 的环境变量

1. 在 ConfigMap 里将环境变量定义为键值对：

   ```shell
   kubectl create configmap special-config --from-literal=special.how=very 
   ```

1. 将 ConfigMap 里的`special.how`值赋给 Pod 的环境变量`SPECIAL_LEVEL_KEY`.  

   ```shell
   kubectl edit pod dapi-test-pod
   ```

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: dapi-test-pod
   spec:
     containers:
       - name: test-container
         image: gcr.io/google_containers/busybox
         command: [ "/bin/sh", "-c", "env" ]
         env:
           # Define the environment variable
           - name: SPECIAL_LEVEL_KEY
             valueFrom:
               configMapKeyRef:
                 # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
                 name: special-config
                 # Specify the key associated with the value
                 key: special.how
     restartPolicy: Never
   ```

1. 将修改保存到 Pod 的配置里，现在 Pod 的输出会包含`SPECIAL_LEVEL_KEY=very`这么一行. 



### 从多个 ConfigMap 读取数据来定义 Pod 环境

1. 跟前面的例子一样，我们先创建 ConfigMap。

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: special-config
     namespace: default
   data:
     special.how: very
   ```

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: env-config
     namespace: default
   data:
     log_level: INFO
   ``` 

1. 在 Pod 配置里定义环境变量.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: dapi-test-pod
   spec:
     containers:
       - name: test-container
         image: gcr.io/google_containers/busybox
         command: [ "/bin/sh", "-c", "env" ]
         env:
           - name: SPECIAL_LEVEL_KEY
             valueFrom:
               configMapKeyRef:
                 name: special-config
                 key: special.how
           - name: LOG_LEVEL
             valueFrom:
               configMapKeyRef:
                 name: env-config
                 key: special.type
     restartPolicy: Never
   ```
 
1. 保存 Pod 的配置，现在 Pod 的输出会包括`SPECIAL_LEVEL_KEY=very`和`LOG_LEVEL=info`.



## 将所有 ConfigMap 的键值对都设置为Pod的环境变量

注意: 这个功能只适用于使用 Kubernetes 1.6及以上版本的环境.

1. 创建一个包含多个键值对的 ConfigMap。

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: special-config
     namespace: default
   data:
     SPECIAL_LEVEL: very
     SPECIAL_TYPE: charm
   ```

1. 使用`env-from`来读取 ConfigMap 的所有数据作为Pod的环境变量。ConfigMap 的键名作为环境变量的变量名。
   
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: dapi-test-pod
   spec:
     containers:
       - name: test-container
         image: gcr.io/google_containers/busybox
         command: [ "/bin/sh", "-c", "env" ]
         envFrom:
         - configMapRef:
             name: special-config
     restartPolicy: Never
   ```

1. 保存 Pod 的配置，现在 Pod 的输出会包含`SPECIAL_LEVEL=very`和`SPECIAL_TYPE=charm`.



## 在 Pod 命令里使用 ConfigMap 定义的环境变量

我们可以利用`$(VAR_NAME)`这个 Kubernetes 替换变量，在 Pod 的配置文件的`command`段使用 ConfigMap 定义的环境变量。

例子如下:

下面的 Pod 配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special_level
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special_type
  restartPolicy: Never
```

在容器 `test-container` 里产生这样的输出：

```shell
very charm
```



## 添加ConfigMap数据到卷

就像这篇文章[如何使用 ConfigMap 配置容器](/docs/tasks/configure-pod-container/configmap.html)介绍的,当您使用 ``--from-file`` 创建 ConfigMap 时，
文件名将作为键名保存在 ConfigMap 的 `data` 段，文件的内容变成键值。

下面的例子展示了一个名为 special-config 的 ConfigMap 的配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.level: very
  special.type: charm
```



### 从 ConfigMap 里的数据生成一个卷

在 Pod 的配置文件里的 `volumes` 段添加 ConfigMap 的名字。
这会将 ConfigMap 数据添加到 `volumeMounts.mountPath` 指定的目录里面（在这个例子里是 `/etc/config`）。
`command` 段引用了 ConfigMap 里的 `special.level`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
  restartPolicy: Never
```

Pod 运行起来后, 执行这个命令 (`"ls /etc/config/"`) 将产生如下的输出：

```shell
special.level
special.type
```



### 添加 ConfigMap 数据到卷里指定路径

使用 `path` 变量定义 ConfigMap 数据的文件路径。
在我们这个例子里，`special.level` 将会被挂载在 `config-volume` 的文件 `/etc/config/keys` 下.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh","-c","cat /etc/config/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.level
          path: keys
  restartPolicy: Never
```

Pod 运行起来后，执行命令(`"cat /etc/config/keys"`)将产生下面的结果：

```shell
very
```



### 绑定 Key 到指定的路径和文件权限

我们可以基于文件来绑定 Key 到指定的路径和文件权限，详情请查阅 [Secrets](/docs/concepts/configuration/secret#using-secrets-as-files-from-a-pod)
这篇文章解释了这个用法。

### 自动更新挂载的 ConfigMap

当一个已经被使用的 ConfigMap 发生了更新时，对应的 Key 也会被更新。Kubelet 会周期性的检查挂载的 ConfigMap 是否是最新的。
然而，它会使用本地基于 ttl 的 cache 来获取 ConfigMap 的当前内容。因此，从 ConfigMap 更新到 Pod 里的新 Key 更新这个时间，等于 Kubelet 的同步周期加 ConfigMap cache 的 tll。

{% endcapture %}

{% capture discussion %}



## 了解 ConfigMaps 和 Pods

### 限制

1. 我们必须在 Pod 使用 ConfigMap 之前，创建好 ConfigMap（除非您把 ConfigMap 标志成"optional"）。如果您引用了一个不存在的 ConfigMap，
那这个Pod是无法启动的。就像引用了不存在的 Key 会导致 Pod 无法启动一样。

1. 如果您使用`envFrom`来从 ConfigMap 定义环境变量，无效的 Key 会被忽略。Pod可以启动，但是无效的名字将会被记录在事件日志里(`InvalidVariableNames`). 
日志消息会列出来每个被忽略的 Key ，比如：

   ```shell
   kubectl get events
   LASTSEEN FIRSTSEEN COUNT NAME          KIND  SUBOBJECT  TYPE      REASON                            SOURCE                MESSAGE
   0s       0s        1     dapi-test-pod Pod              Warning   InvalidEnvironmentVariableNames   {kubelet, 127.0.0.1}  Keys [1badkey, 2alsobad] from the EnvFrom configMap default/myconfig were skipped since they are considered invalid environment variable names.
   ```



1. ConfigMaps 存在于指定的 [命名空间](/docs/user-guide/namespaces/).则这个 ConfigMap 只能被同一个命名空间里的 Pod 所引用。

1. Kubelet 不支持在API服务里找不到的Pod使用ConfigMap，这个包括了每个通过 Kubectl 或者间接通过复制控制器创建的 Pod，
不包括通过Kubelet 的 `--manifest-url` 标志, `--config` 标志, 或者 Kubelet 的 REST API。（注意：这些并不是常规创建 Pod 的方法）
   
{% endcapture %}

{% capture whatsnext %}
* 想了解更多 [ConfigMaps](/docs/tasks/configure-pod-container/configmap/).
* 想研究一个现实生活的案例 [使用ConfigMap配置redis](/docs/tutorials/configuration/configure-redis-using-configmap/).

{% endcapture %}

{% include templates/task.md %}

