# 校园地图与路网数据

本章以 `feat/campus-directory` 的 `049a32064f6368d14b602590c57be3df0057f7e3` 为复现基线。地图数据最后一次修改是 `275606194313ff3004fdd38a726309372477c63d`，包含 2026-11-22 补录的唐岛湾通讯录地点。Android 运行时只读取随包分发的一个 UTF-8 JSON 文件，地图、路网、地点及检索均可离线复现。关联的通讯录、课表由宿主把地点标识或文字交给地图。独立地图无需读取课表、通讯录原始业务数据或学校账号。

## 原始资产与完整性

- 仓库原路径 `feature/campusmap/src/main/assets/campus_map.json`。
- 文件实际大小 **561,670 字节**。SHA-256 为 `76547E26F88B5701EE6BB77E1892363225750BE32580C9768AFE62BC437905E9`。
- 直接使用交接包内的原文件。保留原始数组顺序和坐标精度。搜索同级命中保留地点顺序，路网解析以节点顺序建立稳定索引。将节点 ID 转为整数、对边自动反向或按字典序重排均会改变复现输入或等价路线排序。
- `CampusMapDataSource` 在 IO 调度器上一次读取并解析，Android 版在内存保留结果。项目中没有找到随源码提交的地图生成器、原始 OSM 下载缓存或可无损重建修订点位的脚本。OSM 边界查询是来源记录，动态重拉的要素、处理后的图结构和项目补注点位可能与当前资产不同。

## JSON 契约

| 位置 | 类型与约束 |
| --- | --- |
| 根 | `{version, attribution, dataLicense, licenseUrl, sources, campuses}`。`version` 必须是 `1`，`campuses` 恰好含 `tdw` 和 `gzk` 两个不重复校区。其余四项保留来源与署名，Android 数据解析器不把它们用于算路。 |
| `campuses[]` | `{id, name, bounds, areas, geometry, places, network}`。`id` 是字符串 `tdw` 或 `gzk`，`name` 为显示名。 |
| `bounds` | 四个数 `[west, south, east, north]`，满足 `west < east`、`south < north`。用于地图取景和坐标投影，实际校区覆盖由 `areas` 决定。 |
| `areas[]` | 点数组，每个多边形至少三个点，当前三条边界均首尾同点。包含关系取任一多边形命中。古镇口两块边界相隔，不能拿一个外包矩形把校外区域视为校内。 |
| `geometry[]` | `{kind, points}`。本资产含 `area`、`building`、`field`、`path`、`road`、`water`。它们是显示几何，`building` 同时参与楼内判断。路线应使用 `network`，不能仅凭底图的 `path/road` 线条重建通行关系。 |
| `places[]` | 必填 `id, name, kind, aliases, confidence`。`id` 和分类字段是字符串，`aliases` 是字符串数组，允许为空。`point`、`accessPoint` 是可选坐标，`parentId`、`region` 是可选字符串。地点 ID 在校区内唯一。外部寻址时同时传 `campusId + placeId`。 |
| `network.nodes[]` | `{id, point}`。节点 ID 是不可重复的字符串，可能看起来像 OSM 的数字 ID。`point` 必填。 |
| `network.edges[]` | `{from, to, walk, bike, name?}`。端点引用同校区节点 ID，`walk` 和 `bike` 是布尔通行标记，端点顺序表示允许方向。可选 `name` 是资产附加信息，Android 图模型不读取。 |

所有坐标统一是 WGS84 **`[经度, 纬度]`**，经度范围 ±180，纬度范围 ±90。距离以米为单位。Web 地图投影与 C++ 几何均用双精度接收。如果选用浏览器 Geolocation，需把浏览器具名字段按 `[coords.longitude, coords.latitude]` 组装。若另接采用 GCJ-02 或 BD-09 的底图，应在系统边界显式按来源转换，全程保留资产原坐标系。当前复现可用同一资产直接绘图，无需额外底图服务。

`places[].osmObject`、`previousName` 和 `verification` 属于来源与修订元数据。`verification` 中可能有核实日期 `checkedOn`、核实精度 `precision`、公开来源 `sources` 和说明 `note`。这些字段不控制选路，应原样保留。`point` 缺失时仍可搜索该地点并报告「位置待核」，不要给默认原点。

### 地点、楼栋和入口

- `parentId` 用于把同栋服务、机构和入口关联到主楼栋。一个地点可以有独立标注点，选路时优先用主楼栋的已接路入口。地图基本点层只绘制有坐标、无 `parentId` 且非入口的地点。用户仍可搜索同楼服务。选中某处时优先取该处可用入口作为地图焦点，其次取该处第一个直接关联入口，最后用地点自己的坐标。
- `point` 是展示位置，入口的 `point` 是门的位置。`accessPoint` 是与路网衔接的坐标。两者都有坐标、置信度为 `surveyed`、均在校区多边形内的 `entrance` 才进入可接路候选。唐岛湾还要求门的位置不落在建筑轮廓内。入口 ID 和 `parentId` 都要保留，偏航时才有原入口可继续选择。
- `hasMappedPosition` 接受 `osm`、`verified`、`verified-area`。按地址精确跳转到地图还要求目标有 `point` 且非入口。`verified-building` 服务项可用关联主楼栋算路，区域核实项的终点只能说明到该区域附近。`estimated` 用于位置待核。现有 UI 也识别 `reference` 为参考图说明，当前资产没有这个取值。
- 没有可接路入口的普通目的地，使用可信的主楼栋 `point` 作为道路吸附目标。没有可接路数据的入口自身保持无可用目标。导航图层会区分建筑内部和楼外道路。校区多边形边缘容差是 0.7 米，建筑轮廓判断是 0.05 米。
- 地点搜索按名称、类别、片区和别名匹配，依次优先精确名称、其他键精确、前缀、包含。归一化使用小写并去空白、`·()（）-`。地图搜索列表不展示入口。课表地点有单独的 NFKC 归一化和候选消歧，不能用普通搜索的第一条代替。

## 数据规模校验

| 指标 | 唐岛湾 `tdw` | 古镇口 `gzk` |
| --- | --- | --- |
| `bounds` | `[120.1629774,35.9357862,120.1786518,35.949732]` | `[120.0066226,35.7595551,120.0316627,35.7737038]` |
| `areas` 和各自点数 | 1 个，76 点 | 2 个，15／29 点 |
| 显示几何和几何点总数 | 598 条，4,938 点 | 98 条，1,140 点 |
| 几何分类计数 | area 1，building 90，field 38，path 351，road 108，water 10 | area 2，building 35，field 2，path 23，road 36 |
| 地点记录 | 187，含 98 个非入口与 89 个入口，186 条有 `point`，86 条有 `accessPoint` | 29，东区 19／西区 10，均有 `point`，无入口标注 |
| 地点分类 | entrance 89，food 3，gate 3，landmark 8，library 2，medical 1，residence 22，service 20，sport 18，teaching 21 | food 3，gate 2，medical 1，residence 16，sport 1，teaching 6 |
| `confidence` 数量 | osm 63，verified 10，verified-area 5，verified-building 19，surveyed 89，estimated 1 | osm 29 |
| 路网节点 | 1,580 | 43 |
| 有向边总数和各方式 | 3,656，步行 3,656，骑行 1,756 | 102，步行和骑行均为 102 |
| 仅看边是否存在时的弱连通分量 | 步行节点数 1,565／11／2／2。骑行 798／11／7／6／2／2，另有 754 个节点对骑行孤立 | 步行及骑行均为 39／4 |
| 缺少反向副本的边 | 骑行 32 条，步行 0 条 | 均为 0 条 |

以上是从生产原数据计算出的计数，弱连通不是任意有向节点对均可达的证明。边端点均存在，无自环、相同坐标端点或重复有向端点对。地点的非空父 ID 都能在其校区找到。导入后可按此核对解析器。2026-11-20 验收资料记载的唐岛湾 92 个可搜索点，是后续补录之前的数量。

## 可复核的公开点位

| 校区 | 地点 ID | 名称 | 原始坐标 `[经度, 纬度]` |
| --- | --- | --- | --- |
| tdw | `south-gate` | 南门 | `[120.1734442,35.9386709]` |
| tdw | `library` | 图书馆 | `[120.169302,35.9427139]` |
| tdw | `south-building` | 南教楼 | `[120.1724811,35.9413987]` |
| tdw | `foundation-lab-d` | 基础实验楼D区 | `[120.1702364,35.943784]` |
| tdw | `yifu-building` | 逸夫楼 | `[120.171988,35.9393928]` |
| tdw | `employment-center` | 学生就业指导中心 | `[120.1706854,35.9400255]` |
| tdw | `training-building` | 培训楼 | `[120.176709,35.9409257]` |
| gzk | `teach` | 东区综合教学楼 | `[120.0295987,35.7694618]` |
| gzk | `east-sport` | 东区文体中心 | `[120.0251245,35.7704055]` |
| gzk | `osm-w1359338293` | 东区观海苑餐厅 | `[120.0265875,35.7704283]` |
| gzk | `osm-n12977523907` | 东区校医院 | `[120.0245466,35.7699135]` |
| gzk | `west-food` | 西区珠山餐厅 | `[120.0161452,35.7665128]` |
| gzk | `west-dorm` | 西区学8 | `[120.012751,35.7648781]` |

「东区／西区」是 `region`，表中加前缀是为了易读，原始 `name` 无此前缀。不要从示例把片区名称拼接到地点名。逸夫楼、学生就业指导中心和培训楼的父 ID 分别为 `auditorium`、`dorm-19`、`training`。课表入口应验证「南教110」命中 `south-building`、「微601」和「微（６０１）教室」命中 `foundation-lab-d`、纯房号和冲突校区不命中。通讯录入口应验证 `{campusId:"tdw", placeId:"yifu-building"}` 精确选中，并以父楼栋的已接路入口为候选。

`stadium-tennis`（网球场〔看台东侧〕）目前只有检索记录，无坐标。`sport-entry-1`（体育馆东南门）、`lecture-group-entry-55`（讲堂群西北门）、`liberalarts-entry-31`（文理楼西南门）有标注坐标而缺 `accessPoint`。古镇口东区可沿已录入的道路计算路线，西区地物与搜索可用，但路网稀疏。历史算路验收中，西区 `west-food → west-dorm` 步骑均无可用路线。路线不应跨校区或穿过未连接的片区、临时通行限制和楼门开放状态以现场为准。上述数据校验不是现场走测结果。

## 来源、署名与数据更新

- 资产根字段是 `attribution = "© OpenStreetMap contributors"`、`dataLicense = "ODbL-1.0"`、`licenseUrl = "https://www.openstreetmap.org/copyright"`。Web 地图可见位置保留相同署名与许可入口，并随原始公开路网数据保留许可元数据。项目的 MIT `LICENSE` 用于软件源码和文档，地图数据使用资产注明的许可。 [OpenStreetMap 版权和数据许可](https://www.openstreetmap.org/copyright)
- 唐岛湾保存的公开 OSM 来源范围为 `120.163,35.9358,120.1786,35.9496`，古镇口为 `120.004,35.754,120.034,35.775`，两个 URL 原样记录在 JSON 的 `sources`。路网是从地图数据整理后的有向边，`bounds` 是渲染用包围范围。
- 唐岛湾入口来自 2026-11-20 的用户公开楼门标注，按 OSM 建筑轮廓微调门的位置并核对路网短连接。2026-11-22 又由地图标注和学校公开办事地址补录通讯录定位点。公开核实文档与精度范围详见数据各项 `verification` 及原项目 `docs/adr/007-offline-campus-navigation.md`，使用地图时可保留点位来源。
- 更新数据时要继续保存稳定校区、地点和节点引用，核对边方向和步骑限行，以楼栋轮廓复核入口，按固定校区多边形验证可达性。古镇口西区或其他缺路处，先补经核实的道路资料再开放路线。地图数据更新不要求接入任何外部在线服务。发布分发时应再次核对当前数据许可。
- 不导出个人轨迹、精确位置历史、课表正文、真实账号或会话。浏览、搜索、计算和位置匹配可全部在设备会话内存完成。浏览器如另行加载远端瓦片或让后端计算实时路线，会改变当前的数据流，需要单独设计。

原实现依据为 `CampusMapData.kt` 的数据模型、`GeoPoint`、`CampusMapDataSource` 和地点匹配，`CampusMapController.kt` 的课表／通讯录选择，`CampusMapCanvas.kt` 的绘制与地名锚点，以及 `docs/adr/007-offline-campus-navigation.md`、`docs/PRIVACY.md`。
