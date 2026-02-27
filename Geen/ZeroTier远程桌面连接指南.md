1.官网注册登录账号并下载及安装客户端
https://www.zerotier.com/org/
<img width="1663" height="838" alt="image" src="https://github.com/user-attachments/assets/2708eb91-9315-4c13-9c21-96be7c3c0997" />


2.启动客户端后，创建群组，建立连接
<img width="407" height="89" alt="image" src="https://github.com/user-attachments/assets/74171ee0-7d27-4886-ac35-02de8e8c557b" />
在Join ZeroTier Network里输入上图16位id，连接成功后在控制台同意授权此设备
成功结果如图，操纵节点，被操纵节点（win）都按此方式添加连接
<img width="1465" height="76" alt="image" src="https://github.com/user-attachments/assets/c41d0e8b-a154-43a5-81c9-7c908a235f27" />


购买搭建好云服务器后，先在网络与安全组中新增9993端口规则
<img width="868" height="780" alt="image" src="https://github.com/user-attachments/assets/879f8343-881b-4432-ba80-63bf0b795ecb" />



在云服务器上创建ZeroTier Moon节点，以下是详细的搭建步骤：

一、准备工作

服务器要求：
• 具备固定公网IP的云服务器（推荐阿里云ECS、腾讯云轻量应用服务器）

• 开放UDP 9993端口（在云服务商安全组和服务器防火墙中放行）

• ZeroTier版本需≥1.2.4

二、安装ZeroTier

使用官方脚本一键安装：
curl -s https://install.zerotier.com | sudo bash


启动服务并设置开机自启：
systemctl start zerotier-one.service
systemctl enable zerotier-one.service


三、加入ZeroTier网络

执行以下命令加入你的网络（替换为实际网络ID）：
sudo zerotier-cli join xxxxxxxx


然后登录ZeroTier官网（https://my.zerotier.com），在网络管理页面授权该服务器加入网络

四、配置Moon节点

1. 生成Moon配置文件
cd /var/lib/zerotier-one
sudo zerotier-idtool initmoon identity.public > moon.json


2. 编辑配置文件
vim moon.json


找到 stableEndpoints 字段，修改为（替换为你的公网IP）：
"stableEndpoints": ["你的公网IP/9993"]


3. 生成Moon签名文件
sudo zerotier-idtool genmoon moon.json


4. 部署Moon文件
mkdir moons.d
mv 000000xxxxxxxxxx.moon moons.d/


5. 重启服务
systemctl restart zerotier-one


五、客户端配置

Linux客户端：
# 方法1：复制.moon文件到moons.d目录
mkdir -p /var/lib/zerotier-one/moons.d
cp 000000xxxxxxxxxx.moon /var/lib/zerotier-one/moons.d/
systemctl restart zerotier-one

# 方法2：使用orbit命令
zerotier-cli orbit moon节点ID moon节点ID


Windows客户端：
1. 创建目录：C:\ProgramData\ZeroTier\One\moons.d\
2. 将.moon文件复制到该目录
3. 按Win+R，输入services.msc，找到ZeroTier One服务并重启

六、验证连接

执行以下命令查看节点状态：
zerotier-cli listpeers


如果看到类似以下输出，表示Moon节点配置成功：

200 listpeers
xxxxxxxxxx 你的公网IP/9993;xxx;xxx xxx 1.x.x MOON


七、注意事项

1. 端口开放：确保云服务器安全组和服务器防火墙都开放了UDP 9993端口
2. 公网IP：必须使用服务器的公网IP，不能使用内网IP
3. 版本要求：ZeroTier版本需≥1.2.4才能正常使用Moon功能
4. 延迟优化：建议选择离用户地理位置较近的云服务器区域部署Moon节点，以获得最佳性能



八、强制指定moon模式的参数设置（cmd或powershell执行）
zerotier-cli set 633e31d8a25293a9 allowGlobal=1
zerotier-cli set 633e31d8a25293a9 allowManaged=1
zerotier-cli set 633e31d8a25293a9 allowDefault=0

然后重启服务
net stop ZeroTierOneService
net start ZeroTierOneService

九、连接远程桌面
Ctrl + R 进入运行
输入mstsc运行
输入ZeroTier虚拟IP，用户名和密码（对方密码不能为空，需提前设置密码），点击连接则成功连接远程桌面
<img width="603" height="356" alt="image" src="https://github.com/user-attachments/assets/27b13697-77c6-4188-83a1-0ebdf139da7d" />





