# VPN traffic redirect to another VPN tunnel
VPN的隧道转接技术，实现一个隧道里面的流量直接重定向到另一个隧道。  
实验是我在CentOS 6.5 x86_64上做的，现在已经投入到生产环境中。  
网络一直很稳定，带10个人一起玩东南亚CSGO毫无压力。 
拓扑是这样的：  

http://ww3.sinaimg.cn/large/6f1310d0jw1epxdb762gaj20og0dgwfp.jpg

其他发行版的Linux可以把命令翻译下，原理是一模一样的。  

需要工具：  
1、OpenVPN （UDP封装速度快，效率高。玩游戏必备。PPTP VPN也行，但是速度没那么快。）  
2、ShadowVPN （虽然是Beta状态，使用的效率还是很高的。因为无状态，所以不被干扰。）  
3、IP rule  
4、iptables  

步骤：  

------先做好基本环境配置------  

0、更新生产环境（因为我是在纯净的系统上做的）  

sudo yum -y groupinstall "Development Tools"  
yum -y install openssl*  

1、打开ipv4转发功能。  

vim /etc/sysctl.conf  
net.ipv4.ip_forward = 0 修改成 net.ipv4.ip_forward = 1  
sysctl -p  

2、添加epel源，直接从源安装openvpn，让用户通过openvpn接入国内中转服务器。  

wget http://mirrors.ustc.edu.cn/epel/6/x86_64/epel-release-6-8.noarch.rpm  
rpm -ivh epel-release-6-8.noarch.rpm  
yum makecache  
yum -y install openvpn  

签发证书的教程：  

http://blog.chinaunix.net/uid-29746173-id-4351133.html  

做完后记得打开openvpn使用的端口。  

3、安装ShadowVPN，使用ShadowVPN将国内中转服务器和国外服务器对接起来  

项目地址： https://github.com/clowwindy/ShadowVPN/  

wget https://github.com/clowwindy/ShadowVPN/releases/download/0.1.6/shadowvpn-0.1.6.tar.gz  
tar zxvf shadowvpn-0.1.6.tar.gz  
./configure --enable-static --sysconfdir=/etc  
make && sudo make install  

记得打开ShadowVPN用的端口，不然隧道起不来。  

记得清除掉ShadowVPN client_up.sh中的一段命令：  
 
echo changing default route  
if [ pppoe-wan = "$old_gw_intf" ]; then  
route add $server $old_gw_intf  
else  
route add $server gw $old_gw_ip  
fi  
route del default  
route add default gw 10.7.0.1  
echo default route changed to 10.7.0.1  

不然ShadowVPN up后，你所有的流量都从ShadowVPN隧道走了。  

测试隧道通信是否成功：  
sudo route add -host 8.8.8.8 dev tunX （tunX是你ShadowVPN的interface）  
然后nslookup twitter.com 8.8.8.8  
如果返回的地址是无污染的IP，说明隧道已经UP了。  

OpenVPN这块，如果签发证书后接入成功，且能ping通网关地址，就没问题。  

------再开始做流量的对接------  

我这里的ShadowVPN interface IP使用的是10.20.0.1（境外服务器），10.20.0.2（境内服务器）  
OpenVPN使用的IP段是10.200.0.0/24，请各位写iptables规则的时候替换成你们正在使用的IP段。  

1、首先先定义自定义的路由表，不然写路由条目的时候会写到主路由表，那就会导致选路错乱了。  

vim /etc/iproute2/rt_tables  
添加一行  
200 netgamesg   
然后保存，退出。  
编号是可以自己自定义的，名字也是可以自定义的，我这里为了好记所以写了这个。  
（编号不可以和系统路由表编号重复）  

2、添加静态默认路由。  

ip route add default dev tunX （你的ShadowVPN interface） table netgamesg  

这里是在netgamesg这张路由表里面添加一个默认路由，默认网关出口是shadowvpn的接口。  

3、给用户接入进来的地址打上标记，然后强制打上标记的数据使用netgamesg这张路由表。  

iptables -A PREROUTING -t mangle -s 10.200.0.0/24 -j MARK --set-mark 3  
ip rule add fwmark 3 table netgamesg  

4、使用ip rule来根据源地址来使用路由表。  

ip rule add from 10.200.0.0/24 table netgamesg  

这里是规定源地址为10.200.0.0/24的IP数据包使用netgamesg这张路由表。  

5、最后一步，设置iptables转发。  

iptables -t nat -A POSTROUTING -s [你的openvpn源地址段] -j SNAT --to-source [你的ShadowVPN interface IP]  

6、刷新路由表。  

ip route flush cache  

7、添加脚本，让它开机启动自动执行即可。  

vim route.sh  

shadowvpn -c /etc/shadowvpn/client.conf -s start  
ip route add default dev tun10 table netgamesg  
ip rule add fwmark 3 table netgamesg  
ip rule add from 10.200.0.0/24 table netgamesg  

这样就大功告成了。拨入中国的服务器，你访问ip.cn会显示的是外国的IP，原理就是中国服务器无条件将你的数据转发到国外服务器了。  

这个模型里面不需要考虑MTU的问题，至少我这边做了3次是没出现因为MTU问题导致的数据不通。如果数据不通的话看看哪一步没做对。  

如果有什么地方写的不对，敬请批评指正。  
