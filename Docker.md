# Docker

## 常用指令

```bas
docker push nginx #从docker仓库获取一个nginx镜像
docker images # 查看docker里面有哪些镜像
```

![image-20210306214508475](D:\学习笔记\image-20210306214508475.png)

![image-20210306214638945](D:\学习笔记\image-20210306214638945.png)

```bash
docker run -d -p 80:80 nginx
运行nginx服务器
-d是指后台运行
-p是来指定内外端口映射  80是内 80是外 相当于内和外相同映射
```

