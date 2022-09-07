

# UBI 镜像列表







## UBI-flask

> https://github.com/major/ubi-flask.git









# ubi， ubi-init，ubi-minimal区别



## UBI

```
oc create deployment ubi8 --image registry.access.redhat.com/ubi8/ubi:latest -- sleep 10d

oc create deployment ubi8-root --image registry.access.redhat.com/ubi8/ubi:latest -- sleep 10d

# 创建 sa 并赋予anyuid 权限
oc create sa root-user
oc adm policy add-scc-to-user anyuid -z root-user
oc patch deployment/ubi8-root --patch '{"spec":{"template":{"spec":{"serviceAccountName": "root-user"}}}}'

# 登陆到每一个pod，执行以下命令

id 



```



## UBI-init



UBI init 镜像（名为 `ubi-init）` 包含 systemd 初始化系统，这有助于构建您要在其中运行 systemd 服务的镜像，如 web 服务器或文件服务器。init 镜像内容小于您使用标准镜像获得的内容，但要比最小镜像中的内容要多。



```
# 创建测试pod
oc create deployment ubi8-init-httpd --image quay.io/junkai/ubi8-init-httpd:latest

oc create deployment ubi8-init-httpd-root --image quay.io/junkai/ubi8-init-httpd:latest

# 创建 sa 并赋予anyuid 权限
oc create sa root-user
oc adm policy add-scc-to-user anyuid -z root-user
oc patch deployment/ubi8-init-httpd-root --patch '{"spec":{"template":{"spec":{"serviceAccountName": "root-user"}}}}'



```







# ubi anyuid 模式下的特殊性







