
## 📌 Redis GEO 核心要点

### 1. 基础概念

- **GEO**​ 是 Redis 专门用于处理地理位置数据的数据类型
    
- 底层使用 **Sorted Set**​ 实现，通过 GeoHash 算法将二维经纬度转换为一维分数存储
    
- 所有 GEO 命令本质上都是对 Sorted Set 操作的封装
    

### 2. 重要命令详解

#### 1. GEOADD - 添加地理空间位置

**语法：**​ `GEOADD key longitude latitude member [longitude latitude member ...]`

|参数|含义|规则与示例|
|---|---|---|
|`key`|存储位置的 Sorted Set 的键名|例如：`cities`, `drivers`|
|`longitude`|**经度**（-180 到 180）|东经为正数，西经为负数。例如：北京的经度是 **116.405285**​|
|`latitude`|**纬度**（-85.05112878 到 85.05112878）|北纬为正数，南纬为负数。例如：北京的纬度是 **39.904989**​|
|`member`|地理位置名称|例如：`"北京"`, `"driver:1001"`|

**示例：**

```
# 添加单个位置
GEOADD cities 116.405285 39.904989 "北京"
# 添加多个位置（原子操作）
GEOADD cities 121.4737 31.2304 "上海" 113.2644 23.1291 "广州"
```

#### 2. GEOPOS - 获取地理空间的坐标

**语法：**​ `GEOPOS key member [member ...]`

|参数|含义|规则与示例|
|---|---|---|
|`key`|位置的键名|同上|
|`member`|要查询的位置名称|可查询多个，返回顺序与查询顺序一致|

**示例：**

```
GEOPOS cities "北京" "广州"
# 返回：1) 1) "116.40528291463851929" 2) "39.90498822209174963"
#      2) 1) "113.26443880796432495" 2) "23.12908114130838733"
```

#### 3. GEODIST - 计算两个位置之间的距离

**语法：**​ `GEODIST key member1 member2 [unit]`

|参数|含义|规则与示例|
|---|---|---|
|`key`|位置的键名|同上|
|`member1`|位置1的名称||
|`member2`|位置2的名称||
|`unit`|**可选**，距离单位|**m**（米，默认值）、**km**（千米）、**mi**（英里）、**ft**（英尺）|

**示例：**

```
GEODIST cities "北京" "上海" km
# 返回："1066.9403" （单位：公里）
```

#### 4. GEORADIUS / GEOSEARCH - 查找指定半径内的位置（**推荐使用 GEOSEARCH**）

`GEORADIUS`在 Redis 6.2 后被 `GEOSEARCH`取代，语法更清晰。

**GEOSEARCH 语法：**​ `GEOSEARCH key <FROMMEMBER member | FROMLONLAT longitude latitude> <BYRADIUS radius | BYBOX width height> [ASC|DESC] [COUNT count] [WITHCOORD] [WITHDIST] [WITHHASH]`

|参数|含义|规则与示例|
|---|---|---|
|`key`|位置的键名|同上|
|`FROMMEMBER member`|**二选一**，从指定成员的位置开始搜索|例如：`FROMMEMBER "北京"`|
|`FROMLONLAT longitude latitude`|**二选一**，从给定的经纬度开始搜索|例如：`FROMLONLAT 116.40 39.90`|
|`BYRADIUS radius`|**二选一**，按圆形半径搜索|例如：`BYRADIUS 100 km`|
|`BYBOX width height`|**二选一**，按矩形框搜索|例如：`BYBOX 200 150 km`（一个200km x 150km的矩形）|
|`ASC|DESC`|**可选**，按距离中心点的远近排序|
|`COUNT count`|**可选**，限制返回结果数量|例如：`COUNT 10`|
|`WITHCOORD`|**可选**，在结果中返回经纬度||
|`WITHDIST`|**可选**，在结果中返回距离中心点的距离||
|`WITHHASH`|**可选**，在结果中返回原始的 GeoHash 值（高级用法）||

**示例：**

```
# 查找北京100公里内最近的两个城市，并返回距离和坐标
GEOSEARCH cities FROMMEMBER "北京" BYRADIUS 100 km ASC COUNT 2 WITHDIST WITHCOORD

# 从经度116.4，纬度39.9开始，搜索一个200km x 200km的矩形框内的所有位置，由远到近排序
GEOSEARCH cities FROMLONLAT 116.4 39.9 BYBOX 200 200 km DESC WITHCOORD
```

#### 5. GEOHASH - 获取地理位置的 GeoHash 值

**语法：**​ `GEOHASH key member [member ...]`

|参数|含义|规则与示例|
|---|---|---|
|`key`|位置的键名|同上|
|`member`|位置名称||

**示例：**

```
GEOHASH cities "北京"
# 返回：1) "wx4g0b7xrt0" 
# 这是一个Base32编码的字符串，可以在地图上快速定位到一个大致区域。
```

---

### 💎 核心要点总结

- **经纬度顺序**：Redis GEO 严格遵守 **(经度, 纬度)**​ 的顺序，与地图API的常见顺序一致，切勿搞反。
    
- **单位是参数的一部分**：在 `GEODIST`和 `GEOSEARCH`中，单位（`km`, `m`等）是直接写在数值后面的，例如 `100 km`。
    
- **推荐使用 GEOSEARCH**：它是 `GEORADIUS`的现代替代品，功能更强大（支持矩形搜索），语法更清晰。
    
- **结果选项**：`WITHCOORD`（返回坐标）、`WITHDIST`（返回距离）等选项可以组合使用，让你只获取需要的数据，提高效率。
    

希望这份详细的参数说明能帮助您彻底掌握 Redis GEO 命令的使用！如果您有任何疑问，请随时提出。