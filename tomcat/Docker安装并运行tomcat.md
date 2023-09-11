1. 先拉tomcat官方镜像
```shell
docker pull tomcat
```

2. 第一次拉起tomcat
	- `8888:8080`，8888端口映射到容器里的8080端口
	- `-v`参数，指定挂载本机的webapps目录
```shell
docker run -d -p 8888:8080 --name tomcat-webapps -v /Users/caicai/Documents/Workspace/Stash/tomcat-webapps:/usr/local/tomcat/webapps tomcat:latest
```

3. 第二次拉起tomcat
如果已经用步骤二执行过了，那么会创建出一个容器，当你修改了webapps里的内容之后，可以通过以下命令启动/重启这个容器：
```shell
docker restart tomcat-webapps
```