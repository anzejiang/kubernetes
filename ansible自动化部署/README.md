# Kubernetes v1.16 高可用集群自动部署（离线版）
>### 确保所有节点系统时间一致（包括ansible节点）
### 1、下载所需文件

下载Ansible部署文件：

```
git clone https://github.com/anzejiang/kubernetes.git
cd ansible自动化部署
```

下载软件包并解压：

云盘地址：链接：https://pan.baidu.com/s/1H5ClTXPkYAyzFPAqGdSFqA 
提取码：t96b

解压至/root目录下

```
tar zxf binary_pkg.tar.gz
```
### 2、修改Ansible文件

修改hosts文件，根据规划修改对应IP和名称。

```
vi hosts
```
修改group_vars/all.yml文件，修改软件包目录和证书可信任IP。

```
vim group_vars/all.yml
software_dir: '/root/binary_pkg'
...
cert_hosts:
  k8s:
  etcd:
```
## 3、一键部署
### 架构图
单Master架构
![avatar](https://github.com/anzejiang/kubernetes/blob/master/images/single-master.jpg)
多Master架构
![avatar](https://github.com/anzejiang/kubernetes/blob/master/images/multi-master.jpg)

### 部署命令
单Master版：
```
ansible-playbook -i hosts single-master-deploy.yml -uroot -k
```
多Master版：
```
ansible-playbook -i hosts multi-master-deploy.yml -uroot -k
```

## 4、部署控制
如果安装某个阶段失败，可针对性测试.

例如：只运行部署插件
```
ansible-playbook -i hosts single-master-deploy.yml -uroot -k --tags addons
```

### 最后啰嗦一会
group_vars/all.yaml说明：
  定义全局变量文件，其中证书授权IP地址尽量预留一点为新增node节点做准备，
  部署HA集群时，注意网卡名称

另外，add-node.yml是新增node节点所需plabook文件
