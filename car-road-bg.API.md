# carNav 接口文档 · 车辆位置接入

面向后端 / 前端对接人员。动画页面是一个**单文件 HTML**（`car-road-bg.html`，约 114 KB，canvas 2D 实现，含胶片后处理 / 沥青颗粒，无任何外部依赖、可离线运行），所有能力通过全局对象 `window.carNav` 暴露。

---

## 1. 引入方式

```html
<!-- 方式一：iframe 嵌入 -->
<iframe id="car" src="car-road-bg.html"
        style="width:100%;height:100%;border:0;display:block"></iframe>

<!-- 方式二：直接作为整页背景（把 body 内嵌进 .stage 即可） -->
```

页面内部自带 2:1 的舞台容器，会随窗口自适应缩放，不需要额外传参。

---

## 2. 坐标系约定

- 入参统一为 **经度 lng、纬度 lat**，十进制，WGS-84（GPS 原始值）。
- 如果后端拿到的是 **GCJ-02（高德/腾讯）** 或 **BD-09（百度）**，请先转换成 WGS-84 再传入，否则会整体偏移几百米。
- 页面里的中国地图是**简化示意底图**（省级边界抽稀，等经纬投影，未做墨卡托/TM 投影），只用于表达“车开到哪了”，不用于精确测距或导航。

---

## 3. API 一览

| 方法 | 参数 | 返回 | 说明 |
|---|---|---|---|
| `setPosition(lng, lat)` | `number, number` | `undefined` | **追加**一个轨迹点。与上一个点相差小于 `1e-6` 度时自动跳过；跟随开启时自动居中 |
| `setTrack(list)` | `Array` | `undefined` | **替换**整条轨迹。元素可以是 `[lng, lat]` 或 `{ lng, lat }` |
| `getTrack()` | — | `Array<{lng,lat}>` | 读取当前轨迹副本（不影响页面） |
| `clearTrack()` | — | `undefined` | 清空轨迹，只留地图 |
| `open()` | — | `undefined` | 播放转场：升到车顶 → 出现中国地图（约 4.2 秒） |
| `close()` | — | `undefined` | 收回公路画面（约 3.7 秒） |
| `isOpen()` | — | `boolean` | 地图是否已展开（`view.p > 1.2`） |
| `zoomTo(z)` | `1 ~ 6` | `undefined` | 设置地图缩放倍数 |
| `follow(on)` | `boolean` | `undefined` | 跟随模式：新点进来时地图自动把它居中 |

轨迹容量上限 **300 点**，超出后丢弃最早的。

---

## 4. 典型对接（WebSocket 实时推送）

```js
const frame = document.getElementById('car');

frame.addEventListener('load', () => {
  const nav = frame.contentWindow.carNav;

  // 1) 历史轨迹（后端一次性给）
  nav.setTrack(history.map(p => [p.lng, p.lat]));

  // 2) 跟随最新位置
  nav.follow(true);

  // 3) 转场进地图
  nav.open();

  // 4) 实时位置
  ws.onmessage = (ev) => {
    const { lng, lat } = JSON.parse(ev.data);
    nav.setPosition(lng, lat);
  };
});
```

HTTP 轮询同理，把 `setPosition` 放进 `setInterval` 即可，**建议 1 次/秒**，更快也没有视觉收益（轨迹点会挤在一起）。

---

## 5. 演示与调试参数

| 参数 | 作用 |
|---|---|
| `?demo=1` | 模拟导航系统持续推点（上海一带绕行），用于演示 / 联调 |
| `?p=0` ~ `?p=2` | 直接停在转场的任意进度，`2` 即地图态 |
| `?auto=1` | 打开页面自动播一遍完整转场 |
| `?z=2` | 地图初始缩放倍数 |

例：`car-road-bg.html?demo=1`、`car-road-bg.html?p=2&z=2.5`

---

## 6. 交互（用户侧，无需接口）

- 双击小车：升起进入地图
- 地图内：滚轮缩放（以鼠标为中心）、按住拖拽平移、双击任意处返回公路

---

## 7. 兼容性

Chrome / Edge 90+、Safari 15+、Firefox 90+（依赖 Canvas 2D 与 Pointer Events）。
低版本浏览器会出现地图不显示但不报错的情况，建议直接提示升级。

---

## 8. 已知限制

1. 底图为抽稀后的省级轮廓，没有城市、道路级细节。
2. 没有做经纬度纠偏，也不做跨省路径规划——点与点之间是**直线连接**。
3. 新坐标是直接跳变的，未做轨迹平滑/插值；如果需要平滑移动，请在后端或调用侧自行插值后逐点 `setPosition`。
4. 定位点、红点、连线的尺寸不随地图缩放变化（保持屏幕大小），放大时是地图在变。
