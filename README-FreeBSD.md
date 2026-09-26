# 1. 接口锚点地址（真实分配，供 server / ipfw 放行）
ifconfig lo0 inet6 fd00:d0d0:cafe::1/64 alias     # v6 示例
ifconfig lo0 inet 10.99.0.1/24 alias              # v4 示例

# 2. 池前缀路由（出站绕过邻居解析；生产网卡场景不需要，走默认路由）
route add -inet6 -net fd00:d0d0:cafe::/64 ::1
route add -inet  -net 10.99.0.0/24 127.0.0.1

# 3. ipfw：入站强制本地交付（顺序：先放行已分配地址，再 fwd 池前缀）
kldload ipfw
ipfw add 65000 allow ip from any to any
ipfw add 80  allow ip  from any to 10.99.0.1 in
ipfw add 90  allow ip6 from any to fd00:d0d0:cafe::1 in
ipfw add 100 fwd ::1 ip6 from any to fd00:d0d0:cafe::/64 in
ipfw add 105 fwd ::1 ip6 from any to fd00:d0d0:caff::/64 in
ipfw add 110 fwd 127.0.0.1 ip from any to 10.99.0.0/24 in

# 4. 代理（root；-S SOCKS5，-u/-p 认证，-v IPv4 池，-i 多 IPv6 池）
./http-proxy-ipv6-pool -b 0.0.0.0:51080 -S 0.0.0.0:51081 \
    -i 2001:db8:1::/64,2001:db8:2::/64 -v 192.0.2.0/24 -u user -p pass
