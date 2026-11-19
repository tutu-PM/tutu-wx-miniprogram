# 平台约束与组件复用

小程序不是"小一号的网页"。这份列出**会直接影响界面正确性**的平台事实，以及哪些能力该直接用官方。

---

## 一、优先用官方组件，不要自研

### 容器与内容

`view` `text` `scroll-view` `swiper` `swiper-item` `movable-view` `movable-area`
`image` `icon` `progress` `rich-text` `navigator`

### 表单（最容易自研错的）

`button` `input` `textarea` `switch` `slider` `picker` `picker-view` `radio` `radio-group` `checkbox` `checkbox-group` `form` `label`

**判断标准**：只要有"选择、输入、开关、滑动"这四种交互，先查官方组件。自己写的日期选择器、自己画的下拉框，都会在真机上出问题（滚动惯性、键盘遮挡、无障碍、iOS 与安卓表现不一致）。

### 页面级（别自己画）

- 底部 tab：原生 `tabBar` 配置
- 顶部导航：`navigationBarTitleText` / `navigation-bar`
- 下拉刷新：`onPullDownRefresh` + `enablePullDownRefresh`
- 上拉加载：`onReachBottom`
- 页面属性：`page-meta`

自己画 tabbar 的代价：选中态、图标、切换、返回逻辑全要自己维护，还要处理安全区。除非品牌要求极高，不值得。

### tabBar 的硬限制

- 图标必须是**本地图片**，不能用网络图、不能用字体图标
- 图标尺寸建议 81×81px，大小不超过 40KB
- 最多 5 个 tab

---

## 二、能力必须走 open-type

自己画按钮 = 拿不到平台能力。这是小程序最常见的架构错误。

| open-type | 得到什么 | 自研为什么不行 |
|---|---|---|
| `getPhoneNumber` | 手机号（加密数据） | 无法获取，只能让用户手输 |
| `share` | 拉起转发面板 | 只能靠右上角菜单，无法自定义触发点 |
| `contact` | 客服会话 | 打不开 |
| `openSetting` | 授权设置页 | 用户无法重新授权 |
| `feedback` | 意见反馈页 | 打不开 |
| `launchApp` | 拉起 App | 打不开 |

**正确做法**：外观自研 + 透明原生按钮覆盖。

```xml
<view class="my-share-btn">
  <text>分享给家人</text>
  <button class="cover" open-type="share"></button>
</view>
```

```css
.my-share-btn { position: relative; }
.my-share-btn .cover {
  position: absolute; inset: 0;
  opacity: 0;
}
```

`button` 自身样式要重置：`border: none; background: transparent; padding: 0; line-height: inherit;` 并去掉 `::after` 边框。

---

## 三、三个例外：官方组件不能无条件用

### 例外 1：原生组件层级最高

`map` `video` `canvas` `camera` `live-player` `live-pusher` 由客户端绘制，**永远在页面最上层**，`z-index` 压不住。

后果：
- 在这类组件上弹自绘弹窗 → 弹窗被盖住，只剩原生组件
- 在 `scroll-view` 里放 `video` → 滚动时可能错位

处理：
- 布局阶段就避开：不要把弹层设计在原生组件正上方
- 必须覆盖时用 `cover-view` / `cover-image`（能力受限，只能放文本和图片）
- 或依赖同层渲染（`video` 2.4.0+、`map` 2.7.0+ 已支持，但低版本基础库仍会退化）

### 例外 2：原生外观改不动

`button` 有默认边框和内边距，要重置；`picker` 的弹层外观固定，不能换成自定义样式。

处理：
- 要自定义外观 → 用 `view` 触发 + 官方 API（如 `wx.showActionSheet`）
- 要保留能力 → 接受原生外观，或做透明覆盖

### 例外 3：自绘控件不得绕开 open-type

可以做自定义视觉，但不能因为有"设计感"就把分享、手机号这些能力绕开。**能力走官方，外观自己做**是唯一正确姿势。

---

## 四、适配：不是"选设备"，是处理底线

小程序只有手机端，但仍要处理这四件事：

### 1. 安全区

- 顶部：刘海屏、状态栏
- 底部：iPhone 横条，会压住底部按钮和 tabbar

```css
.footer { padding-bottom: calc(24rpx + env(safe-area-inset-bottom)); }
```

底部固定操作栏、自定义 tabbar 必须处理，否则在 iPhone 上按钮被横条盖住一半。

### 2. 胶囊按钮

右上角胶囊是**微信客户端绘制**的，位置固定、不可改。

- 自绘顶部导航必须避让胶囊（用 `wx.getMenuButtonBoundingClientRect()` 拿位置）
- 顶部标题、右侧操作图标不能放在胶囊区域
- 用户点胶囊区域的下方是可以的，但不要贴着

### 3. 屏幕尺寸跨度大

iPhone SE：375×568
Pro Max：430×932

**首屏可用高度差接近一倍。** 按 SE 算，能放下的东西比你想的少。

- 首屏关键内容按 SE 验证
- 大屏不要靠"拉伸"填满，用留白或增加内容密度
- 不要在首屏放"必须滚动才能看到的主动作"

### 4. iOS / Android 差异

| 现象 | 原因 |
|---|---|
| 字体粗细不同 | 系统字体渲染差异 |
| 滚动回弹不同 | iOS 有弹性，安卓没有 |
| `picker` 外观不同 | 两端原生控件样式不同 |
| 日期格式 | 安卓旧版本 `new Date('2024-01-01')` 可能返回 Invalid Date |
| 图片透明通道 | 部分安卓机渲染异常 |

---

## 五、和 Web 不一样的硬规则

| 项目 | Web 的做法 | 小程序实际 |
|---|---|---|
| 悬停 | `:hover` | 不存在，用 `hover-class` 或 `:active` |
| 滚动 | 页面自然滚动 | 长列表用 `scroll-view`，需要固定头部时必须用 |
| 字体 | 任意网络字体 | **不能任意加载**，包体积也受限，用系统字体 |
| 阴影 / 毛玻璃 | 常用装饰 | 低端安卓掉帧，谨慎 |
| 动画 | 自由 | 优先 `transform` / `opacity`，避免动画 `height` / `top` |
| 选择文字 | 默认可选 | 默认不可选，`text` 加 `user-select` 才行 |
| 富文本 | 直接插 HTML | 用 `rich-text`，且不支持所有标签 |
| 点击事件 | `click` | `bindtap` / `catchtap`；注意 `catch` 阻止冒泡 |
| 页面传参 | URL / state | 只能 URL 拼参数，复杂数据要放全局或存储 |

---

## 六、图片：真机最容易翻车的地方

### 1. 格式

- **不用 WebP。** iOS 真机的 `image` 组件对 WebP 支持不稳定：开发者工具和安卓可能正常，iPhone 显示空白。
- 不透明图用 **JPG**，需要透明用 **PNG**。
- 验证不能只靠开发者工具或 H5 预览——必须真机看。

### 2. 裁切

`image` 的 `mode="widthFix"` 只保证宽度，高度自适应，**但父容器如果写了固定 `height` 加 `overflow: hidden`，图片照样被裁**。

排查顺序：
1. 父容器有没有写死 `height`
2. 有没有 `overflow: hidden`
3. `mode` 是不是 `widthFix`
4. 图片本身比例对不对

### 3. 大图

单张图建议不超过 200KB，列表用缩略图 + 点击看大图（`wx.previewImage`）。

---

## 七、包体积

- **主包上限 2MB**，总包 20MB
- 图片、音视频、大量 JSON 都会吃主包
- 超过就要分包（`subPackages`），或把资源放到 CDN

**从 0 阶段就该注意**：不要把所有图标都塞 `src/static`。图标优先用 `icon` 组件或少量雪碧图。

---

## 八、弹层怎么选

| 需求 | 用什么 |
|---|---|
| 简单提示 | `wx.showToast` |
| 二次确认 | `wx.showModal` |
| 底部选项 | `wx.showActionSheet` |
| 自定义内容弹窗 | 自绘 `view` + 遮罩层 |
| 半屏页面 | `page-container` |

自绘弹窗要处理：滚动穿透（`catchtouchmove`）、点击遮罩关闭、安全区。
