# Portainer

## 可视化

portainer就是docker的图形化界面管理工具，提供一个后台面板供使用。

**安装portainer**：

```shell
docker run -d -p 8088:9000 \
--restart=always -v /var/run/docker.sock:/var/run/docker.sock --privileged portainer/portainer
```

