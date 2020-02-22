---
title: AspNetCore2.1升级到3.1时CORS相关配置的变更
---

本周将公司的运营系统从AspNetCore2.1升级到了AspNetCore3.1,遇到了一些坑，这里记录以下

## 2.1的相关CORS代码


```
public IServiceProvider ConfigureServices(IServiceCollection services)
{

        ... // 省略
        services.AddCors(option => option.AddPolicy("corsPolicy", builders =>
        {
            builders.AllowCredentials().AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod();
        }));
        ... // 省略
}

public void Configure(IApplicationBuilder app, ILoggerFactory loggerFactory, IHostingEnvironment env)
{
    ... // 省略
    app.UseCors("corsPolicy");
     ... // 省略
}
```

如果这部分代码这3.1的服务中不加任何修改，启动时提示错误：


```
The CORS protocol does not allow specifying a wildcard (any) origin and credentials at the same time. Configure the CORS policy by listing individual origins if credentials needs to be supported.
```


这是由于在2.1之后,AspNetCore出于安全考虑，做了更加严格的限制，在不AllowCredentials()与AllowAnyOrigin()。

假设现在站点A存在一个恶意脚本，而站点B存在一个比较的敏感的接口（如转账）。如果站点B作为服务端使用AllowCredentials()与AllowAnyOrigin()的同源配置，此时站点A可以直接调用站点B的敏感接口并发送凭证信息（如Cookie），那么将导致用户信息被窃取。



## 3.1的相关CORS代码

### options1 显示声明允许跨域的origin

```
services.AddCors(option => option.AddPolicy("corsPolicy", builders =>
            {
                builders.WithOrigins("http://site.com").AllowAnyHeader().AllowAnyMethod().AllowCredentials();
            }));
```


### options2 使用 SetIsOriginAllowed


```
services.AddCors(option => option.AddPolicy("corsPolicy", builders =>
            {
                builders.SetIsOriginAllowed(origin=>true).AllowAnyHeader().AllowAnyMethod().AllowCredentials(); //SetIsOriginAllowed(origin=>true)允许所有origin
            }));
```


