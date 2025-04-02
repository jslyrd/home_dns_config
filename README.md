# home_dns_config

# SingBox P核 FakeIP+ikuai网关分流+ADGuardHome缓存去广告+IPv6方案（真-养老配置，稳定运行没再动过）
0. 前提
    - 设备：1、爱快（或其他能自定义流量转发的路由）作为网关；2、旁路网关上有clash（或其他支持fakeip的工具，我这里使用singbox p核）；3、ADGuardHome（非必要）
    - 使用SingBox P核根据ip数据库（使用shellcrash可以自动下载、定时更新所需的所有东西）分别转发国内和非国内域名到不同DNS（国内扔给adg，国外用谷歌的加密dns），注意要用p核，否则部分配置不支持。同时，singbox也自带dns缓存。
    - ADGuardHome主要用作开箱即用的DNS记录（国内）、缓存和去广告，没有需求可以直接用其他公共dns代替。
1. 直接上内网DHCP+DNS设置和流量走向
    - 设置：爱快负责DHCP，宣告网关为爱快(172.16.6.1)，主DNS为SingBox(172.16.6.9:53)，备用DNS为阿里的223.6.6.6
    - 流量走向：
        1. 内网机器（客户端）上网时首先向SingBox(172.16.6.9:53)发起DNS查询，由于是内网，不会经过爱快
        2. SingBox根据数据库将国内域名和其他域名分开，国内域名交给ADGuardHome，会返回正常ip；其他域名SingBox直接查询谷歌的加密dns并直接返回FakeIP；
        3. 客户端拿到ip后发起连接，来到网关(爱快)这里，爱快静态路由将Fakeip就扔给SingBox，不是就直接发出去，完成连接
        5. SingBox收到FakeIP会路由出去，此时已经换成真实ip并加密流量了，爱快也会直接放行。
      
2. 优点：
    1. 内网用户无感使用，不用分别设置网关，国内流量也不经过旁路网关，直接走爱快。
        - 这对于旁路网关是虚拟机或docker是很大的优势，不会像常见的旁路由那样造成爱快和旁路各处理一次
    2. 其他流量走FakeIP，速度起飞。
    3. ipv6全程无影响，理论上SingBox的套装支持ipv6的话也能出去，已验证
    4. 抗风险，旁路网关挂了也不影响国内流量，怎么折腾都没事。
        - 对于客户端来说，只有主选DNS服务器无响应时备用dns才会发挥作用，主选DNS服务器不存在的域名解析不会去使用备选DNS服务器进行解析。旁路网关挂掉时dns肯定不正常，此时客户端会使用爱快下发的备用dns，爱快只转发fakeip给clash，所以旁路网关挂了是不会影响的
    5. DNS记录、缓存、去广告等，可以用ADGuardHome，也可能直接去掉ADGuardHome，随便用一两个国内加密DNS即可。
    6. 流程清晰，配置一次永久使用。IP数据库有ShellCrash自动更新，且即使ip数据库不再维护，也不影响旧的数据库使用
3. 配置项
    1. 爱快及旁路网关：
        1. 网络设置-DHCP服务端-网关设置为爱快的地址，主DNS设置为旁路网关地址，备选DNS设置为阿里的223.6.6.6或腾讯之类的国内DNS，其他随意
            ![image](https://github.com/user-attachments/assets/e919a2ae-9382-4a5c-b2cc-46f896633869)
        2. 网络设置-静态路由-添加以下规则即可（注意规则冲突时是后添加的先生效），看图：
            ![image](https://github.com/jslyrd/home_dns_config/blob/main/%E9%9D%99%E6%80%81%E8%B7%AF%E7%94%B1%E9%85%8D%E7%BD%AE.jpg)
    2. SingBox：
        1. 这里用的是shellcrash标准linux设备安装：`export url='https://fastly.jsdelivr.net/gh/juewuy/ShellCrash@master' && wget -q --no-check-certificate -O /tmp/install.sh $url/install.sh  && bash /tmp/install.sh && source /etc/profile &> /dev/null`
        2. 只需设置纯净模式+mix模式，DNS进阶不用管会被dns.json覆盖，我使用了
        3. 可以小抄一下作业，下面的配置开启了fakeip、dns缓存，没有使用shellcrash的路由拦截，因为有爱快的静态路由和tun拦截了，非常地纯净。如果有不想走fakeip的域名，将其放在domain_suffix中即可：
```
# 以linux标准安装方式下载shellclash，选项时装在/etc，使用sing-box puernya的内核。设置防火墙为nftables，开启ipv6，纯净模式+mix模式，DNS进阶不用管会被dns.json覆盖。
export url='https://fastly.jsdelivr.net/gh/juewuy/ShellCrash@master' && wget -q --no-check-certificate -O /tmp/install.sh $url/install.sh  && bash /tmp/install.sh && source /etc/profile &> /dev/null
cat > /etc/ShellCrash/jsons/experimental.json << EOF
{
  "experimental": {
    "clash_api": {
      "external_controller": "0.0.0.0:9999",
      "external_ui": "ui",
      "secret": "",
      "default_mode": "Rule"
    },
    "cache_file": {
      "enabled": true,
      "path": "",
      "cache_id": "",
      "store_fakeip": true
    }
  }
}
EOF

cat > /etc/ShellCrash/jsons/inbounds.json << EOF
{
  "inbounds": [
    {
      "type": "tun",
      "tag": "sing-box-tun-in",
      "interface_name": "sing-box-tun",
      "address": [
        "172.16.0.1/16",
        "fdfe:dcba:9876::1/126"
      ],
      "mtu": 9000,
      "auto_route": true,
      "auto_redirect": true,
      "strict_route": true,
      "stack": "system",
      "route_address": [
        "198.18.0.0/15",
        "fc00::/18"
      ],
      "route_exclude_address": [
        "172.16.0.0/16"
      ]
    }
  ]
}
EOF
cat > /etc/ShellCrash/jsons/dns.json << EOF
{
  "dns": {
    "hosts": {
      "doh.pub": [ "1.12.12.12", "120.53.53.53", "2402:4e00::" ],
      "dns.alidns.com": [ "223.5.5.5", "223.6.6.6", "2400:3200::1", "2400:3200:baba::1" ],
      "dns.google": [ "8.8.8.8", "8.8.4.4", "2001:4860:4860::8888", "2001:4860:4860::8844" ],
      "cloudflare-dns.com": [ "1.1.1.1", "1.0.0.1", "2606:4700:4700::1111", "2606:4700:4700::1001" ]
    },
    "servers": [
      { "tag": "dns_direct", "address": "127.0.0.1:1745", "detour": "DIRECT" },
      { "tag": "dns_proxy", "address": [ "https://dns.google/dns-query", "https://cloudflare-dns.com/dns-query" ] },
      { "tag": "dns_fakeip", "address": "fakeip" }
    ],
    "rules": [
      { "outbound": [ "any" ], "server": "dns_direct" },
      { "clash_mode": [ "Direct" ], "query_type": [ "A", "AAAA" ], "server": "dns_direct" },
      { "clash_mode": [ "Global" ], "query_type": [ "A", "AAAA" ], "server": "dns_proxy" },
      { "geosite": [ "cn" ], "query_type": [ "A", "AAAA" ], "server": "dns_direct" },
      { "geosite": [ "proxy" ], "query_type": [ "A", "AAAA" ], "server": "dns_fakeip" },
      { "fallback_rules": [ { "geoip": [ "cn" ], "server": "dns_direct" }, { "match_all": true, "server": "dns_fakeip" } ], "server": "dns_direct" }
    ],
    "final": "dns_proxy",
    "strategy": "prefer_ipv4",
    "independent_cache": true,
    "lazy_cache": true,
    "reverse_mapping": true,
    "mapping_override": true,
    "fakeip": {
      "enabled": true,
      "inet4_range": "198.18.0.0/16",
      "inet6_range": "fc00::/16",
      "exclude_rule": {
        "geosite": [ "fakeip-filter" ],
        "domain_suffix":[
          ".cn",
          "steam-chat.com",
          "steamcommunity.com",
          "cm.steampowered.com",
          "steamdb.info",
          "steamstatic.com"
          ]
      }
    }
  }
}
EOF
```
    3. ADGuardHome：
        1. 下载（先cd到安装目录）：`wget https://static.adguard.com/adguardhome/edge/AdGuardHome_linux_amd64.tar.gz -O AdGuardHome.tar.gz`   
        2. 配置监听的DNS端口与SingBox dns配置的dns_direct要一致，即1745。其他配置不在此赘述
        - 要修改配置：
            1. 停止服务：`/usr/local/AdGuard_Home/AdGuardHome -s stop`
            2. 修改配置：`vim /usr/local/AdGuard_Home/AdGuardHome.yaml`
            3. 启动服务：`/usr/local/AdGuard_Home/AdGuardHome -s start`

- 注：自定义DNS配置部分参考了DustinWin的搭载 sing-boxp 内核进行 DNS 分流教程-geodata 方案[https://proxy-tutorials.dustinwin.top/posts/dnsbypass-singboxp-geodata/]
       
