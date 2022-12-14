### # 配置系统时区为东八区
```
rm -f /etc/localtime
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### # ubuntu官方源安装nginx和依赖包并设置开机启动
```
apt clean all && apt update
apt install nginx curl pwgen openssl netcat cron uuid-runtime -y
systemctl enable nginx
systemctl start nginx
ufw disable
```

### # 开始部署之前，我们先配置一下需要用到的参数，如下：
### # “域名，端口，uuid，ws路径，ssl证书目录“
### # “ngin和v2ray配置文件目录“
```
#1.设置你的解析好的域名，如本例子中的vmess.v2ray.one
domainName="vmess.v2ray.one"

#2.随机生成v2ray需要用到的服务端口
port="`shuf -i 20000-65000 -n 1`"

#3.随机生成一个uuid
uuid="`uuidgen`"

#4.随机生成一个websocket需要使用的path
path="/`pwgen -A0 6 8 | xargs |sed 's/ /\//g'`"

#5.以时间为基准随机创建一个存放ssl证书的目录
ssl_dir="$(mkdir -pv "/usr/local/etc/v2ray/ssl/`date +"%F-%H-%M-%S"`" |awk -F"'" END'{print $2}')"

#6.定义nginx和v2ray配置文件路径
nginxConfig="/etc/nginx/conf.d/v2ray.conf"
v2rayConfig="/usr/local/etc/v2ray/config.json"
```

### # 检测域名解析是否正确
```
#域名解析正确不会输出任何内容，如果不正确会退出当前终端
local_ip="$(curl ifconfig.me 2>/dev/null;echo)"
resolve_ip="$(host "$domainName" | awk '{print $NF}')"
if [ "$local_ip" != "$resolve_ip" ];then echo "域名解析不正确";exit 9;fi
```

### # 使用v2ray官方命令安装v2ray，并设置开机启动
```
curl -O https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh
curl -O https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-dat-release.sh
bash install-release.sh
bash install-dat-release.sh
systemctl enable v2ray
```

### # 安装acme,并申请加密证书
### #会提示安装socat，这里使用alpn模式，不用理会
```
source ~/.bashrc
if nc -z localhost 443;then /etc/init.d/nginx stop;fi
if nc -z localhost 443;then lsof -i :443 | awk 'NR==2{print $1}' | xargs -i killall {};sleep 1;fi
if ! [ -d /root/.acme.sh ];then curl https://get.acme.sh | sh;fi
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt
~/.acme.sh/acme.sh --issue -d "$domainName" -k ec-256 --alpn
~/.acme.sh/acme.sh --installcert -d "$domainName" --fullchainpath $ssl_dir/v2ray.crt --keypath $ssl_dir/v2ray.key --ecc
chown www-data.www-data $ssl_dir/v2ray.*
```

### # 把续签证书命令添加到计划任务
```
echo -n '#!/bin/bash
/etc/init.d/nginx stop
"/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" &> /root/renew_ssl.log
/etc/init.d/nginx start
' > /usr/local/bin/ssl_renew.sh
chmod +x /usr/local/bin/ssl_renew.sh
(crontab -l;echo "15 03 * * * /usr/local/bin/ssl_renew.sh") | crontab
```

### # 配置nginx，执行如下命令即可添加nginx配置文件
```
echo "
server {
	listen 80;
	server_name "$domainName";
	return 301 https://"'$host$request_uri'";

}
server {
	listen 443 ssl http2 default_server;
	listen [::]:443 ssl http2 default_server;
	server_name "$domainName";

	ssl_certificate $ssl_dir/v2ray.crt;
	ssl_certificate_key $ssl_dir/v2ray.key;
	ssl_ciphers EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:"!"MD5;
	ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;

	root /usr/share/nginx/html;
	
	location "$path" {
		proxy_redirect off;
		proxy_pass http://127.0.0.1:"$port";
		proxy_http_version 1.1;
		proxy_set_header Upgrade "'"$http_upgrade"'";
		proxy_set_header Connection '"'upgrade'"';
		proxy_set_header Host "'"$http_host"'";
	}

}
" > $nginxConfig
```

### # 配置v2ray，执行如下命令即可添加v2ray配置文件
```
echo "
{
  "log" : {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },
  "inbound": {
    "port": '$port',
    "listen": "127.0.0.1",
    "protocol": "vmess",
    "settings": {
      "decryption":"none",
      "clients": [
        {
          "id": '"\"$uuid\""',
          "level": 1
        }
      ]
    },
   "streamSettings":{
      "network": "ws",
      "wsSettings": {
           "path": '"\"$path\""'
      }
   }
  },
  "outbound": {
    "protocol": "freedom",
    "settings": {
      "decryption":"none"
    }
  },
  "outboundDetour": [
    {
      "protocol": "blackhole",
      "settings": {
        "decryption":"none"
      },
      "tag": "blocked"
    }
  ],
  "routing": {
    "strategy": "rules",
    "settings": {
      "decryption":"none",
      "rules": [
        {
          "type": "field",
          "ip": [ "geoip:private" ],
          "outboundTag": "blocked"
        }
      ]
    }
  }
}
" > $v2rayConfig
```

### # 默认配置vmess协议，如果指定vless协议则配置vless协议
```
[ "vless" = "$2" ] && sed -i 's/vmess/vless/' $v2rayConfig
```

### # 完工，你现在只需要重启v2ray和nginx即可
```
systemctl restart v2ray
systemctl status -l v2ray
/usr/sbin/nginx -t && systemctl restart nginx
```

### # 输出配置信息
```
echo
echo "域名: $domainName" 
echo "UUID: $uuid" 
[ "vless" = "$2" ] && echo "协议：vless" || echo "额外ID: 0" 
echo "安全: tls"
echo "传输: websocket"
echo "路径: $path"
```

### # 注：为适应v2ray-core目前和未来的版本更新，vmess协议额外ID选项已移除，客户端vmess协议额外id配置为0即可，当前最新版v2rayN客户端已默认为0

### #网络参数优化
```
echo "
fs.file-max = 1000000
fs.inotify.max_user_instances = 8192
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_rmem = 16384 262144 8388608
net.ipv4.tcp_wmem = 32768 524288 16777216
net.core.somaxconn = 8192
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.wmem_default = 2097152
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_max_syn_backlog = 10240
net.core.netdev_max_backlog = 10240
net.ipv4.tcp_slow_start_after_idle = 0
# forward ipv4
net.ipv4.ip_forward = 1
">>/etc/sysctl.conf
```
