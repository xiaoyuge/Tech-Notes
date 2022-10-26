DNS服务（域名解析到 IP）：
http 请求的dns请求过程：浏览器缓存 -> 本地操作系统缓存 -> local dns server缓存（跟小区网络运营商相关） -> root dns server（.） -> 各级权威域名服务器（com./cn./org. -> xxx.com. -> yyy.xxx.com. ）-> name dns sever（域名注册服务商的服务器，返回ip地址）
一个域名下可以配置多条不同的A记录，此时 name dns sever可以根据自己的策略来进行选择，典型的应用是智能线路：根据访问者所处的不同地区（譬如华北、华南、东北）、不同服务商（譬如电信、联通、移动）等因素来确定返回最合适的A记录，将访问者路由到最合适的数据中心，达到智能加速的目的

DNS服务过程示例（假设访问网站icyfenix.cn）\
![DNS](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E4%BA%92%E8%81%94%E7%BD%91%E7%BD%91%E7%BB%9C%E6%9E%B6%E6%9E%84/resources/DNS.png)


A记录 vs cname
A记录：用来制定域名对应的 ip 地址，可以将多个域名解析到一个 ip 地址，但不能将一个域名解析到多个 ip 地址（做负载均衡的时候配置同一个域名到多个ip的多条A记录）
cname：别名解析，可以为一个域名设置一个或多个别名
NS记录：为某个域名指定 DNS解析服务器，也就是指定由哪个 DNS 服务器去解析