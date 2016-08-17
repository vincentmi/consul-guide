# 使用Consul

Consul集群的每个节点都必须先安装Consul.安装非常容易,Consul发布为所支持的平台和架构的二进制包.这个指南不包含从源代码编译Consul的内容.

## 安装Consul

安装Consul,找到适合你系统的包下载他.Consul打包为一个'Zip'文件.

下载后解开压缩包.拷贝Consul到你的PATH路径中,在Unix系统中```~/bin```和```/usr/local/bin```是通常的安装目录.根据你是想为单个用户安装还是给整个系统安装来选择.在Windows系统中有可以安装到```%PATH%```的路径中.

### OS X ###

如果你使用```homebrew```作为包管理器,你可以使用命令 

```sh
brew install consul
```

来进行安装.

## 验证安装




