## HDM安装系统

### 准备java环境

省略，找个java exe 安装

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/hdm-kvm-jnlp.jpg)

#### 添加站点信任

在java控制面板添加站点信任，然后就可以使用kvm agent远程了。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/java-添加安全站点.jpg)



### 挂载iso盘

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/hdm-kvm-mount-iso.png)



### 安装系统

添加热键F7，进入boot menu

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/hdm-boot-menu.png)

选择挂载的iso镜像，开始安装

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/hdm-boot-munu-select.png)

选择virtual cdromo

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/hdm-select-virtual-cd.png)

### 注意点

安装过程中，不能断网，因为HDM安装系统时，服务器是通过网络加载了本地的iso镜像，如果安装图中断网，相当于读取安装文件失败，系统自然安装失败了。