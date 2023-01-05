# 基于Echarts的百度地图叠加arcgis server的WMS图层服务
## 前言
前阵子利用`echarts`+`百度地图`做系统的门户首页，遇到一个要地图上叠加产业城影响范围示意图的需求。查阅文档之后，发现百度地图API确实提供了叠加自定义图层的方法，详情请看：
[百度地图API的Map类](http://lbsyun.baidu.com/cms/jsapi/reference/jsapi_webgl_1_0.html#a0b0)
  
其核心类是TileLayer，详细内容如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/26/17114a3da2850958~tplv-t2oaga2asx-image.image)

同时，百度地图API官网也提供了示例: [添加清华校园微观图](http://lbsyun.baidu.com/jsdemo.htm#g0_2)

但是官网提供的示例根本无法满足我的要求，后面查阅相关文档才发现，百度地图是支持WMS图层服务的，在此，感谢`CodingSir`大佬的博文，给了我不少的启发。链接地址：[baidu地图API叠加自定义图层（一）](https://blog.csdn.net/educast/article/details/71692979)

## 实现思路
### 引入基于Echarts的百度地图
```
<script type="text/javascript" src="https://www.echartsjs.com/zh/dist/echarts-gl.min.js"></script>
<script type="text/javascript" src="https://www.echartsjs.com/examples/vendors/echarts-stat/ecStat.min.js"></script>
<script type="text/javascript" src="https://www.echartsjs.com/examples/vendors/echarts/extension/dataTool.min.js"></script>
<script type="text/javascript" src="https://echartsjs.com/examples/vendors/echarts/map/js/china.js"></script>
<script type="text/javascript" src="https://echartsjs.com/examples/vendors/echarts/map/js/world.js"></script>
<script type="text/javascript" src="https://www.echartsjs.com/examples/vendors/echarts/extension/bmap.min.js"></script>
<script type="text/javascript" src="http://api.map.baidu.com/api?v=3.0&ak=(your ak)></script>
```

### 坐标转换
我们都知道，只有同一坐标系的图层叠加，才不会有误差。所以要叠加`WMS图层服务`，我们还是需要进行坐标转换的。

我的`WMS图层服务`是`百度墨卡托投影坐标系`的，因此我只需要把数据转换为`百度地理坐标系`即可实现地图的叠加。转化的代码是：
```javascript
//百度墨卡托坐标-》百度经纬度坐标
new BMap.Pixel(minx, miny);
``` 

### WMS服务叠加标准
`Web地图服务（WMS）`利用具有地理空间位置信息的数据制作地图。其中将地图定义为地理数据可视的表现。  
这个规范定义了三个操作：
- GetCapabitities返回服务级元数据，它是对服务信息内容和要求参数的一种描述；
- GetMap返回一个地图影像，其地理空间参考和大小参数是明确定义了的；
- GetFeatureInfo（可选）返回显示在地图上的某些特殊要素的信息。

我采用的是GetMap操作，关于GetMap参数，可以从GeoServer官网上查阅
( [WMS服务叠加标准](https://docs.geoserver.org/stable/en/user/services/wms/reference.html#getmap) )，详细内容如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/26/17114de9daf219d5~tplv-t2oaga2asx-image.image)

### 关键代码
```javascript
function addWMSLayer() {
    // 获取百度地图实例
    var map = echarts.init(document.getElementById("echarts")).getModel().getComponent('bmap').getBMap();
    // 初始化地图切片
    var tileLayer = new BMap.TileLayer({ isTransparentPng: true });
    // 百度地图切片方案
    var resolutions = [262144, 131072, 65536, 32768, 16384, 8192, 4096, 2048, 1024, 512, 256, 128, 64, 32, 16, 8, 4, 2, 1];
    // 获取切片地址
    tileLayer.getTilesUrl = function (tileCoord, zoom) {
        var x = tileCoord.x;
        var y = tileCoord.y;

        var res = resolutions[zoom];
        var tileWidth = 256;
        var minx = x * tileWidth * res;
        var miny = y * tileWidth * res;
        var maxx = (x + 1) * tileWidth * res;
        var maxy = (y + 1) * tileWidth * res;
        
        var bottomLeft = new BMap.Pixel(minx, miny); // 左下角坐标
        var topRight = new BMap.Pixel(maxx, maxy); // 右上角坐标
        var projection2 = map.getMapType().getProjection();
        var bottomLefXY = projection2.pointToLngLat(bottomLeft); //百度墨卡托坐标-》百度经纬度坐标
        var topRightXY = projection2.pointToLngLat(topRight); //百度墨卡托坐标-》百度经纬度坐标
        var bbox = [bottomLefXY.lng, bottomLefXY.lat, topRightXY.lng, topRightXY.lat]; //计算出bbox

        //根据geoserver WMS服务的规则设置URL
        var url = wmsTileLayer(your wms serverURL) + "?request=GetMap&version=1.3.0&styles=&transparent=true&format=image/png32&layers=0&CRS=CRS:84&WIDTH=256&HEIGHT=256&BBOX=" + bbox.join(',');
        return url;
    };
    map.addTileLayer(tileLayer);
}
```
### 效果展示
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/15/1717d3099e586bf6~tplv-t2oaga2asx-image.image)

#### 注意：
```
如果没有设置transparent=true这个参数，会导致百度地图底图会被wms服务图层覆盖。
```