

# http proxy

http 代理将一个http请求转发到代理服务器。

# socks 协议

socks 一个位于会话层的代理协议。

client <---> proxy server <---> remote

# shadowsocks proxy

client <---> ss-local <--[encrypted]--> ss-remote <---> target

shadowsocks 基于socks协议实现，图中的 ss-local 就是socks协议中的proxy server。

# References

[https://imququ.com/post/web-proxy.html](https://imququ.com/post/web-proxy.html)
[rfc1928](https://tools.ietf.org/html/rfc1928)
[https://thorns.cn/2018/05/06/%E8%B0%88%E8%B0%88socks5%E5%8D%8F%E8%AE%AE%E5%92%8Css.html](https://thorns.cn/2018/05/06/%E8%B0%88%E8%B0%88socks5%E5%8D%8F%E8%AE%AE%E5%92%8Css.html)
[https://guiyongdong.github.io/2017/12/09/Socks5%E4%BB%A3%E7%90%86%E5%88%86%E6%9E%90/](https://guiyongdong.github.io/2017/12/09/Socks5%E4%BB%A3%E7%90%86%E5%88%86%E6%9E%90/)
[ref1](https://bingtaoli.github.io/2016/11/23/shadowsocks%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86/)
[ref2](http://bingtaoli.github.io/2017/05/07/shadowsocks%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86-%E4%BA%8C/)
[ref3](http://hengyunabc.github.io/something-about-science-surf-the-internet/)
