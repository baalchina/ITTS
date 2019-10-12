# 为什么是OpenResty？
* OpenResty是国人开发的基于Nginx的服务器，由于内置了Lua的支持，因此要比Nginx更加灵活。
* OpenResty支持nginx的配置文件，如果你原来就使用了Nginx作为反向代理，那么配置文件拷贝过去就可以了。
* 由于Lua的支持，可以实现一些Nginx原本难以实现的功能，例如：定时断网


# 安装OpenResty

OpenResty的安装很简单，



# 使用lua-rest-auto-ssl来实现SSL证书的自动部署

* 在OpenResty中我使用的是`lua-resty-auto-ssl`来实现证书的部署
* `lua-resty-auto-ssl`的方式是通过动态接收https请求，自动向`let's encrypt`请求，并自动完成证书部署。也就是说，在你允许的情况下，只要向特定服务器发送一个https请求，例如：`curl https://www.baidu.com`，那么OpenResty就会自动调用插件完成证书的部署。这个过程对于管理员和用户来说都是无感知的。
