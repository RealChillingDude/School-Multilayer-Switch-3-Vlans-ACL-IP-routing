# School-Multilayer-Switch-3-Vlans-ACL-IP-routing
1. 核心总结 (大局观)
这个网络的核心在于那台 Multilayer Switch (ML_school)。它不仅是交换机，也是路由器。
角色： 它是所有 VLAN（教师、学生、访客）的网关 (Gateway)，也是通往互联网的桥梁。
关键点： VLAN 之间的数据流转、DHCP 请求的中继、以及安全访问控制（ACL），全部都在这台设备上完成。
最大的坑： 它的路由功能默认是关闭的，必须手动开启。

2. 完整且正确的配置指令 (ML_school)
这是你可以直接复制进去的“满分答案”。我已经把所有的逻辑（路由、VLAN、ACL、端口）都整合好了。
第一阶段：全局与基础设置
enable
configure terminal
! 进入特权模式并进入全局配置模式

hostname ML_school
! 修改设备名称，方便识别

ip routing
! 【最关键指令】开启三层交换机的路由功能。没有它，VLAN之间通不了。

enable secret class
! 设置特权模式加密密码为 "class"

line console 0
 password cisco
 login
 exit
! 设置 Console 线的登录密码为 "cisco"
第二阶段：配置上行接口 (连接路由器)
interface GigabitEthernet0/1
 no switchport
 ! 关闭交换功能，将接口变为“路由口”，这样才能直接配 IP
 
 ip address 10.100.0.2 255.255.255.252
 ! 配置连接到 Router_school 的 IP 地址
 
 no shutdown
 ! 激活接口
 exit
第三阶段：创建 VLAN 并分配端口
! 创建 VLAN 数据库
vlan 10
 name docenten
exit
vlan 20
 name studenten
exit
vlan 30
 name gasten
exit

! 分配教师组端口 (包括打印机、AP、服务器)
interface range f0/1 - 9, f0/19, f0/21, f0/24
 switchport mode access
 switchport access vlan 10
 exit

! 分配学生组端口
interface range f0/10 - 18, f0/20, f0/22
 switchport mode access
 switchport access vlan 20
 exit

! 分配访客端口
interface f0/23
 switchport mode access
 switchport access vlan 30
 exit
第四阶段：配置网关 (SVI) 与 DHCP 中继
interface Vlan10
 ip address 10.1.0.1 255.255.255.0
 no shutdown
 ! 教师 VLAN 的网关 IP。Server 在这里，不需要中继。
 exit

interface Vlan20
 ip address 10.2.0.1 255.255.255.0
 ip helper-address 10.1.0.2
 no shutdown
 ! 学生 VLAN 网关。Helper 指向服务器 IP，帮助获取 DHCP。
 exit

interface Vlan30
 ip address 10.3.0.1 255.255.255.0
 ip helper-address 10.1.0.2
 no shutdown
 ! 访客 VLAN 网关。Helper 指向服务器 IP。
 exit
第五阶段：默认路由 (通往互联网)
ip route 0.0.0.0 0.0.0.0 10.100.0.1
! 告诉交换机：所有不知道去哪的数据包（比如去 Google 的），都扔给路由器 10.100.0.1
第六阶段：ACL 安全策略 (阻止学生访问老师)
这里采用老师那种更稳健的写法。

! 1. 定义规则列表
ip access-list extended BLOCK_STUDENTS
 permit udp any host 10.1.0.2 eq bootps
 ! 解释：允许任何人(any)访问服务器(10.1.0.2)的 DHCP 端口(67/bootps)。
 ! 这一条必须在最前面，否则学生拿不到 IP。
 
 deny ip any 10.1.0.0 0.0.255.255
 ! 解释：拒绝任何人访问 10.1.x.x 开头的整个网段。
 ! 0.0.255.255 是通配符掩码，表示后两位随意，前两位必须匹配。
 
 permit ip any any
 ! 解释：允许剩下的所有流量（主要是去互联网的）。
 exit

! 2. 将规则应用到接口
interface Vlan20
 ip access-group BLOCK_STUDENTS in
 ! 解释：在 VLAN 20 的“入口”方向应用。一旦学生数据进入交换机，立刻检查。
 exit
 
3. 需要特别注意的点 (避坑指南)
虽然上面是标准答案，但在排错时，请务必检查以下 3 个死穴：

1. 路由器的“回程路由” (Return Route)
这是最容易被遗忘的！交换机配置得再完美，如果路由器 (Router_school) 不知道怎么回来，你也上不了网。
现象： 交换机能 Ping 通互联网，但笔记本电脑不行。
解决： 你必须去 Router_school 上加这几条静态路由：
Bash
Router(config)# ip route 10.1.0.0 255.255.255.0 10.100.0.2
Router(config)# ip route 10.2.0.0 255.255.255.0 10.100.0.2
Router(config)# ip route 10.3.0.0 255.255.255.0 10.100.0.2

2. ip routing 必须开
再次强调，多层交换机如果不输这个命令，它就是个“哑巴”，VLAN 之间完全隔离。

3. DHCP Pool 的网关设置
在配置服务器 (Server1) 的 DHCP 服务时，Default Gateway 不能填服务器自己的 IP，也不能填路由器的 IP。
必须填对应 VLAN 的网关 IP（即交换机上 interface vlan X 的那个 IP）。
Docenten Pool 网关 -> 10.1.0.1
Studenten Pool 网关 -> 10.2.0.1
Gasten Pool 网关 -> 10.3.0.1
