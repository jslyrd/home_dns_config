# home_dns_config

# MosDNS DNS分流+Clash FakeIP+ikuai网关分流+ADGuardHome缓存去广告+IPv6方案（养老配置了）
0. 前提
    - 设备：1、爱快（或其他能自定义流量转发的路由）作为网关；2、旁路网关上有clash（或其他支持fakeip的工具）；3、ADGuardHome和MosDNS（非必要）
    - MosDNS作用是分别转发国内和非国内域名到不同DNS，虽然clash本身也能干这活，但是我用的shellcrash测试是有点问题的，主要是部分域名他在规则里识别出是国内ip，但返回的却仍是fakeip，可能是域名不在国内域名列表但ip在国内ip列表。前置一个MosDNS不会出现这种问题，MosDNS-CN的分流模式十分强大且开箱即用，详情可看：[https://github.com/IrineSistiana/mosdns-cn?tab=readme-ov-file#%E5%88%86%E6%B5%81%E6%A8%A1%E5%BC%8F]
    - ADGuardHome主要用作开箱即用的DNS记录、缓存和去广告，没有需求可以直接用其他公共dns代替。
1. 直接上内网DHCP+DNS设置和流量走向
    - 设置：爱快负责DHCP，宣告网关为爱快(172.16.6.1)，主DNS为MosDNS(172.16.6.9:53)，备用DNS为阿里的223.6.6.6
    - 流量走向：内网机器（客户端）上网时首先发起DNS查询->MosDNS根据数据库将国内域名和其他域名分开，国内域名交给ADGuardHome，会返回正常ip；其他域名交给Clash，会返回FakeIP；->客户端拿到ip后发起连接，来到网关(爱快)这里，爱快端口分流，是Fakeip就扔给clash，不是就直接发出去，完成连接->clash收到FakeIP会路由出去，此时已经换成真实ip并加密流量了，爱快也会直接放行。
   ![未命名绘图 drawio (1)](https://github.com/user-attachments/assets/585e0083-078d-4021-948d-5005aca3aee6)
      
2. 优点：
    1. 内网用户无感使用，不用分别设置网关，国内流量也不经过旁路网关，直接走爱快。
        - 这对于旁路网关是虚拟机或docker是很大的优势，如果是单纯根据内网客户端分流，国内流量会经过旁路网关（虽然不一定经过clash内核），会造成爱快和旁路各处理一次，对于非专业转发的硬件是不小的消耗（跑bt/pt时带宽占满的时候非常明显）
    2. 其他流量走FakeIP，速度起飞。
    3. ipv6全程无影响，理论上clash的套装支持ipv6的话也能出去，未验证。国内的正常，已验证
    4. 抗风险，旁路网关挂了也不影响国内流量，怎么折腾都没事。
        - 对于客户端来说，只有主选DNS服务器无响应时备用dns才会发挥作用，主选DNS服务器不存在的域名解析不会去使用备选DNS服务器进行解析。旁路网关挂掉时dns肯定不正常，此时客户端会使用爱快下发的备用dns，爱快只转发fakeip给clash，所以旁路网关挂了是不会影响的
    5. DNS记录、缓存、去广告等都可以在MosDNS、ADGuardHome上操作
    6. 流程清晰，配置一次永久使用。只有定期更新ip数据库的脚本是自己叫AI写的，其他全部是开箱即用。即使ip数据库不再维护，也不影响旧的数据库使用
3. 配置项
    1. 爱快及旁路网关：
        1. 网络设置-DHCP服务端-网关设置为爱快的地址，主DNS设置为旁路网关地址，备选DNS设置为阿里、腾讯之类的国内DNS，其他随意
        2. 流控分流-端口分流-添加规则（注意规则冲突时是后添加的先生效）
            1. 添加规则1，选择分流方式为外网线路，线路选择默认出口，其他不用配置。这条是保底规则其他规则匹配不上时直接走直连，必须！
            2. 添加规则2，选择分流方式为下一跳网关，网关写旁路网关，目的地址写`198.18.0.0/16`（clash fakeip默认，可根据实际修改），其他不用配置
            3. 添加规则3，选择分流方式为外网线路，线路选择默认出口，源地址写旁路网关IP，其他不用配置，这条可选。
            - 注意，如果不设置保底规则，clash内存爆了准是代理了所有流量，爱快在匹配不到规则时使用的转发规则真搞不懂
            4. 添加规则4，下一跳网关，旁路网关，协议udp，源地址写非网关内网ip，目的端口53，其他不用配置
            5. 添加规则5，外网线路，默认出口，udp，目的地址写备用dns，目的端口53，其他不用配置
            - 规则4和5可选，目的是拦截dns请求并转发到mosdns。因为有些应用使用的内置dns。转发后要在旁路网关拦截才生效。备用dns不拦截！
            ![image](https://github.com/user-attachments/assets/e919a2ae-9382-4a5c-b2cc-46f896633869)
            ![image](https://github.com/user-attachments/assets/d4a30a79-da2b-4a91-816d-110093427a6f)
           ![image](https://github.com/user-attachments/assets/f688c8ae-7f3a-4cc9-9f17-f671cd711c65)
        - 旁路网关下执行拦截dns（可选）
            - 如果是使用ufw和nftables：
                - 在`/etc/ufw/after6.rules`文件最后加入以下内容并`ufw reload`即可：
                ```
                *nat
                :PREROUTING ACCEPT [0:0]
                :OUTPUT ACCEPT [0:0]
                -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
                -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53
                COMMIT

                ```
            - 如果是使用iptables:
                ```
                iptables -t nat -A PREROUTING -i ens32 -p udp --dport 53 -j REDIRECT --to-ports 53
                iptables -t nat -A PREROUTING -i ens32 -p tcp --dport 53 -j REDIRECT --to-ports 53
                ```
                - 设置开机持久化
                ```
                echo iptables -t nat -A PREROUTING -i ens32 -p udp --dport 53 -j REDIRECT --to-ports 53 >> /etc/profile
                echo iptables -t nat -A PREROUTING -i ens32 -p tcp --dport 53 -j REDIRECT --to-ports 53 >> /etc/profile
                ```
    2. MosDNS：
        1. 下载数据ip和域名数据库文件：
            - `wget https://cdn.jsdelivr.net/gh/Loyalsoldier/v2ray-rules-dat@release/geoip.dat`
            - `wget https://cdn.jsdelivr.net/gh/Loyalsoldier/v2ray-rules-dat@release/geosite.dat`
        2. 下载MosDNS：
            - `wget https://github.com/IrineSistiana/mosdns-cn/releases/download/v1.4.0/mosdns-cn-linux-amd64.zip`
        3. 启动命令（我习惯用docker的vukomir/busybox，这里仅贴出命令）：
            - `mosdns-cn -s :53 --local-upstream 127.0.0.1:1745 --local-domain "geosite.dat:cn" --local-ip "geoip.dat:cn" --remote-upstream 127.0.0.1:1053 --remote-domain "geosite.dat:geolocation-!cn" --blacklist-domain  "geosite.dat:category-ads-all"`
        - 1745是adguardhome监听的dns端口，1053则是clash监听的dns端口
    3. ADGuardHome：
        1. 下载（先cd到安装目录）：`wget https://static.adguard.com/adguardhome/edge/AdGuardHome_linux_amd64.tar.gz -O AdGuardHome.tar.gz`   
        2. 配置监听的DNS端口与MosDNS的本地上游要一致，即1745。其他配置不在此赘述
        - 要修改配置：
            1. 停止服务：`/usr/local/AdGuard_Home/AdGuardHome -s stop`
            2. 修改配置：`vim /usr/local/AdGuard_Home/AdGuardHome.yaml`
            3. 启动服务：`/usr/local/AdGuard_Home/AdGuardHome -s start`
    4. Clash：
        1. 这里用的是shellcrash标准linux设备安装：`export url='https://fastly.jsdelivr.net/gh/juewuy/ShellCrash@master' && wget -q --no-check-certificate -O /tmp/install.sh $url/install.sh  && bash /tmp/install.sh && source /etc/profile &> /dev/null`
        2. 设置DNS运行模式为FakeIP，设置禁用DNS劫持，设置劫持流量范围为局域网+本机。设置基础DNS为adguardhome监听的dns端口（可选）。其他随意，根据实际情况即可。
    
- 其他可选配置:定期自动更新ip数据库
    - `crontab -e`添加以下内容：
        ```
        # 更新MosDNS 数据库
        30 3 * * 6 /root/Docker/frpc_haproxy_lucky/config/mosdns/updategeo.sh > /root/docker/frpc_haproxy_lucky/config/mosdns/updategeo.log
        ```
    - 添加后可`crontab -l`查看，或执行一次看是否成功
    - `/root/Docker/frpc_haproxy_lucky/config/mosdns/updategeo.sh`内容如下：
        ```bash
        #!/bin/bash

        # 定义文件及其URL对
        FILES=(
            "geoip.dat:https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geoip.dat https://cdn.jsdelivr.net/gh/Loyalsoldier/v2ray-rules-dat@release/geoip.dat"
            "geosite.dat:https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geosite.dat https://cdn.jsdelivr.net/gh/Loyalsoldier/v2ray-rules-dat@release/geosite.dat"
        )

        # 下载函数
        download_file() {
            local file_name=$1
            local urls=($2)
            local temp_file=$(mktemp)
            local success=0

            for url in "${urls[@]}"; do
                echo "尝试从 $url 下载 $file_name..."
                if curl -s -L -o "$temp_file" "$url" && [ -s "$temp_file" ]; then
                    echo "$file_name 下载成功"
                    # 覆盖原文件
                    mv "$temp_file" "$file_name"
                    success=1
                    break
                else
                    echo "$url 下载失败或文件为空"
                fi
            done

            if [ $success -eq 0 ]; then
                echo "$file_name 下载失败，所有URL都不可用"
                rm -f "$temp_file"
                return 1
            else
                return 0
            fi
        }

        # 初始化下载成功标志
        all_downloads_successful=1

        # 遍历文件并下载
        for entry in "${FILES[@]}"; do
            IFS=: read -r file_name file_urls <<< "$entry"
            if ! download_file "$file_name" "$file_urls"; then
                all_downloads_successful=0
                break
            fi
        done

        # 根据下载结果决定是否重启 Docker 容器
        if [ $all_downloads_successful -eq 1 ]; then
            echo "所有文件下载成功，重启 Docker 容器 mosdns..."
            docker restart mosdns
            if [ $? -eq 0 ]; then
                echo "Docker 容器 mosdns 重启完成"
            else
                echo "Docker 容器 mosdns 重启失败"
            fi
        else
            echo "脚本执行失败，因为无法下载所有文件"
        fi

        echo "脚本执行完成"
        ```
- 附docker启动mosdns的docker-compose配置
    ```
    mosdns:
        image: vukomir/busybox:latest     # 使用带curl的镜像，解决https证书错误
        container_name: mosdns
        network_mode: host
        environment:
          - TZ=Asia/Shanghai
        volumes:
          - ./config/mosdns:/usr/local/bin
        working_dir: /usr/local/bin
        entrypoint: [""]
        command: 
          - sh
          - -c
          - |
            mosdns-cn -s :53 --local-upstream 127.0.0.1:1745 --local-domain "geosite.dat:cn" --local-ip "geoip.dat:cn" --remote-upstream 127.0.0.1:1053 --remote-domain "geosite.dat:geolocation-!cn" --blacklist-domain  "geosite.dat:category-ads-all"
        logging:
          options:
            max-size: 200k
        restart: always
    ```

       
