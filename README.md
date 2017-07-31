# benlinhuo.github.com

### 域名换回到xxx.github.io
原先域名是：http://benlinhuo.cn， 发现域名过期无法使用了，想恢复到github给定的域名：http://benlinhuo.github.io 进行访问。解决方案：删除CNAME文本内容，然后_config.yml文件中的url更改成http://benlinhuo.github.io 即可。如果访问http://benlinhuo.github.io 的时候，发现还会重定向到原来的域名，则清除下浏览器缓存再刷新下，就OK了！
