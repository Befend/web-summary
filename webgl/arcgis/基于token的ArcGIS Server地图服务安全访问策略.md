# 基于token的ArcGIS Server地图服务安全访问策略
## 一、前言
> 最近在工作中，经常会遇到关于地图服务访问的安全性问题，对于这个问题，我翻阅了很多资料，网上给出的方案大多都是做代理。根据官网给出的指导，在多次尝试之后，并未有满意的结果。最后在与同事的反复讨论之后，确定并实现了解决方案的技术路线 —— **基于用户权限的 ArcGIS 10.2 for Server Rest服务安全性管理**。

## 二、基于用户权限的 ArcGIS 10.2 for Server Rest服务安全性管理
```ArcGIS Server 安全性```确定了管理 GIS 服务器、发布到 GIS 服务器以及使用服务的用户。

### 1. 创建用户
```用户```是指访问 ArcGIS Server 资源的任何人员或者软件代理商。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2531f6c1d96b4f899936dd44633c5c6e~tplv-k3u1fbpfcp-watermark.image)

### 2. 设置地图服务的访问权限
将```地图服务的安全设置```为私有，```仅面向用户```开放访问权限。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/386e77bdb0d744abb21ed2207cf20157~tplv-k3u1fbpfcp-watermark.image)
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c34b414c2ebc458db25a7ed8cffe687b~tplv-k3u1fbpfcp-watermark.image)

### 3. 地图服务访问测试
打开设置好的地图服务地址进行访问：```https://<host>:<port>/<site>/MapServer```，此时，就需要输入```用户的账号和密码```才能进行访问，这样就达到目的了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fce11c63a2c845999fbfe1889b5e97d3~tplv-k3u1fbpfcp-watermark.image)

## 三、基于token的地图服务地址访问认证处理

### 1. 获取地图服务地址访问的token
这里可以使用```ArcGIS REST API```提供的接口来获取token：

> ```http://<host>:<port>/<site>/generateToken```。
  ```接口的详细信息```可以参考：[https://developers.arcgis.com/rest/services-reference/generate-token.htm](https://developers.arcgis.com/rest/services-reference/generate-token.htm)。
  
#### (1) ```客户端```获取token

在```浏览器```上打开```http://<host>:<port>/<site>/generateToken```，输入已经配置好的参数值，即可获取```地图服务地址访问的token```。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e139b4ab7c441a2af9237f29631c9e4~tplv-k3u1fbpfcp-watermark.image)

#### (2) ```前端```获取token的代码

```javascript
/**
 * @description：获取地图服务访问的token
 * @params： url{String} 请求token的地址, 格式如：http://<host>:<port>/<site>/generateToken
 */
function getArcgisToken(url) {
    var params = {
        username: userName, // userName: 用户名
        password: password, // password: 用户密码
        client: 'requestip', // 客户端标识类型
        referrer: '', // 参照方式
        ip: '',
        expiration: 60 * 24 * 10, // 单位：分钟，可以不设置，不设置时默认最大
        f: 'json'
    };
    $.ajax({
        type: "get",
        url: url,
        data: params,
        dataType: "json",
        success: function (data) {
            if (data.success) {
                var arcgisToken = data.data.Token; // 地图服务地址访问的令牌
                ...
            }
        },
        error: function (error) {
            console.log(error);
         }
    });
}
```

#### (3) ```后端```获取token的代码

```java
public class ArcgisServerToken {
    @Value("${arcgis.server.url}")
    private String url; // 请求token的地址
    @Value("${arcgis.server.username}")
    private String username; // 用户名
    @Value("${arcgis.server.password}")
    private String password; // 用户密码
    @Value("${arcgis.server.client}")
    private String client; // 客户端标识类型
    @RequestMapping
    public AjaxResult getToken() throws IOException {
        HttpRequester request = new HttpRequester();
        Map param = new HashMap();
        param.put("username", username); 
        param.put("password", password); 
        param.put("client", client); 
        param.put("referer", "");
        param.put("ip", "");
        param.put("f", "json");
		// param.put("expiration", 60*24*10); // 单位：分钟，可以不设置，不设置时默认最大
        HttpRespons respons = (HttpRespons) request.sendPost(url, param);
        String json = respons.getContent();
        JSONObject jsonObject = JSONObject.fromObject(json);
        String token = jsonObject.getString("Token");
        return new AjaxResult(token);
    }
}
```

### 2. 基于token的地图服务地址的安全访问

通过上述的方式，无论拼接的是```后端```还是```前端```返回的token，均已实现对地图服务地址的安全访问策略。

> ```http://<host>:<port>/<site>/MapServer?token=LM1zdTdqq8heIbGGJk9Iao7h5WIZskdFRpQFARe-UzVOeT_rEJjTR3Nk47VWB1Fu```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1473a0b8336242e29a1a9c4ed81a660f~tplv-k3u1fbpfcp-watermark.image)

## 四、总结

+ 遇到这样的问题，首先查找是否有相关的工作场景的技术方案及实现思路；有技术方案时，通过分析评估看需要怎么样的实现以及可能遇到的问题。

+ 本文的技术实现路线概括为：
	
    1. 在```地图服务管理器(ArcGIS Manager)```创建拥有访问权限的用户；
    2. ```配置```对应地图服务的```访问权限```；
    3. 可以通过```客户端```、```前端```、```后台```获取地图服务访问的token;
    4. 在```浏览器```访问```拼接token后的地图服务```。

+ 做事要趁早，多尝试。早前看到这个问题，查找资料大都是通过代理解决，所以就想着以后可以通过后台搭建代理框架。其实就应该多尝试，毕竟```实践是检验真理的唯一标准```。

## 五、参考文章
博主 [技术小胖子](https://yq.aliyun.com/users/bnch2y3za6tla?spm=a2c4e.11153940.0.0.2a92237fEthH9M) 的文章：[ArcGIS 10.1 for Server Rest服务安全性管理：基于用户和角色权限](https://yq.aliyun.com/articles/514448)

博主 [羊子雄起](https://blog.csdn.net/comeonyangzi) 的文章：[ArcGIS for JavaScript获取token](https://blog.csdn.net/comeonyangzi/article/details/54669867?utm_medium=distribute.pc_relevant.none-task-blog-OPENSEARCH-7.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-OPENSEARCH-7.control)