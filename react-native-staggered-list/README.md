# react-native-staggered-list

基于 `VirtualizedList` 封装的 `react-native` 瀑布流组件。

经过无数次迭代，目前包里面有三个版本 `deprecated`、`default` 和 `withDimensions`。

这里面最开始的版本是可以自动测量高度的 `deprecated`，后来做了一半比较稳定的 `default`，而且经过线上验证了。最近这一版做的是 `withDimensions`。

主要区别如下:

|                | Deprecated | Default    | withDimensions |
| :------------- | :--------- | :--------- | :------------- |
| 环境等级       | 测试 `Dev` | 生产 `Pro` | 生产 `Pro`     |
| 排列方式       | 最短列填充 | 从左到右   | 最短列填充     |
| 自动计算高度   | ✅         | x          | x              |
| 分页           | x          | ✅         | ✅             |
| 滚动到指定位置 | x          | ✅         | ✅             |

- 业务场景:

  实际上数据源 `data` 都是运营的同学负责添加的，现在的 `APP` 基本上滑动不到底部，因为数据量实在很大。所以就算是几个列的高度差别很大，基本也不影响。这也是我决定最后一次改版的主要原因。

- 用户体验:

  1. 之前的版本都是动态计算，那就必然得等前一个渲染完了，才能渲染下一个。还是之前说的问题，会像 鸡 `🐔` 下蛋 `🥚` 一样，一个个的往外蹦。
  2. 新版本我加入了动画，交互效果看起来能好一些。

虽然说

## 😍 组件特色

### 🌽 智能排列

- `ScrollView` → `VirtualizedList`。
- 经过这么多期不断优化迭代，可见部分采用 `从左到右` 依次填充，不可见部分采用 `高度最小列` 优先填充。

### 🍉 泛型支持

像 `FlatList` 一样 `renderItem`，然后支持自己的 `ItemT`。

### 🍇 扩展性强

- 支持自定义列数 `columns`。

- 支持下拉刷新 `onRefresh()`。

- 支持自定义 `Header` 和 `Footer`。

- 支持自定义列表的 `Container` 样式。

- 支持滑动监听 `onScroll(NativeSyntheticEvent<NativeScrollEvent>) => void`。

- 支持分页加载，加载完了一页回调 `onCompleted: () => void`，接着来下一页的数据也是 `OJBK` 的。

**觉得有用，路过的各位老铁们右上角的小星星走起来，谢谢。**

![](https://net-cctv3.oss-cn-qingdao.aliyuncs.com/net.cctv3.open/StaggeredListDemo0215.gif)

## 😌 命名规范

整体的设计思想模仿的是 `FlatList`，提供以下内容的自定义。

| Name                         | Type                                                | Description                                                                     |
| :--------------------------- | :-------------------------------------------------- | :------------------------------------------------------------------------------ |
| columns                      | `number`                                            | 列数。                                                                          |
| datas                        | `ItemT []`                                          | 数据源 ( 支持泛型 ItemT )。                                                     |
| renderItem                   | `(item: ItemT) => React.Node`                       | 跟 `FlatList` 一样，渲染什么您说了算。                                          |
| onLoadComplete               | `() => void`                                        | 数据全部渲染完成时候的回调，比如 `分页` 这种应用场景。                          |
| header                       | `React.Node`                                        | 瀑布流的头部。                                                                  |
| footer                       | `React.Node`                                        | 瀑布流的尾部。                                                                  |
| showsVerticalScrollIndicator | `boolean`                                           | 是否显示 `纵向` 滚动条。                                                        |
| onScroll                     | `(NativeSyntheticEvent<NativeScrollEvent>) => void` | 滑动事件，比如 `吸顶` 要判断滑动距离这种场景。                                  |
| onRefresh                    | `() => void`                                        | 下拉刷新时候的回调。                                                            |
| columnsStyle                 | `StyleProp<ViewStyle>`                              | 瀑布流的 Container 的样式，可以控制 `内边距` 以及 `列表` 和 `Header` 的距离等。 |

## 🤔 如何使用

```bash
npm install react-native-staggered-list
```

```javascript
import {
  Waterfall,
  WaterfallWithDimensions,
} from "react-native-staggered-list";
```

```javascript
<Waterfall
  onRefresh={() => {
    setR(Math.random());
    setPageIndex(1);
  }}
  header={<View />}
  datas={datas}
  renderItem={(item) => <HomeItem item={item} />}
  columns={2}
  columnsStyle={{
    justifyContent: "space-around",
    paddingHorizontal: 5 * vw,
  }}
  onScroll={(e) => {
    setTabBarOpacity(Math.min(1, e.nativeEvent.contentOffset.y / imgHeight));
  }}
  onLoadComplete={() => {
    setPageIndex((t) => t + 1);
  }}
/>
```

## 😳 实现原理

两种思路：

### Waterafall

直接挨个 `index%column` 往里面填充，适合左右两边高度差不多相等的情况。

### WaterfallWithDimensions

需要在数据源中加入 `dimensions: {width: number, height: number}`，然后根据每一列的高度，填充最低的高度。

### Deprecated

不推荐，有很多缺陷。

- `Item` 会不断的 `onLayout()` 还会有硬件方面性能的损失，再就是就算是拿到 `renderItem` 里面的状态的话，那也是像老母鸡 🐔 下蛋 🥚 一样，一个一个的渲染，体验上也说不过去。

- `Item` 的 `onLayout()` 其实并不是预期的那样，他会立即执行一次或者两次，而不是布局变化的时候进行回调。那么我就要在 `renderItem()` 里面做文章，但是 `Item` 好像拿不到 `props.children` 里面的状态，这就很麻烦。想了很多方法，感觉都不是很好。

代码贴出来:

```js
const Item: React.FC<ItemProps> = (props) => {
  return (
    <View
      onLayout={(layout) => {
        // console.log(layout.nativeEvent.layout);
        layout.nativeEvent.layout.height > 0 &&
          props.onMeasuredHeight(layout.nativeEvent.layout.height);
      }}
    >
      {React.isValidElement(props.children) &&
        React.cloneElement(props.children, {
          nextRender: (next: boolean, height: number) => {
            next && props.onMeasuredHeight(height);
          },
        })}
      {/* {props.children} */}
    </View>
  );
};
```

之前想了个办法，刚开始肉眼可见的区域是直接从左到右依次填充。给了一个高度容错的范围，默认 `[0, 2*props.columns]`。在这个范围里面的数据，渲染的时候，延时 `1000ms`，这样儿确保了前面的数据渲染完了，拿到的高度能更真实一些。也就是说最后这几个 `Item` 是优化布局，纠错用的。

但是这样儿分页又出问题了。有可能这一页还没渲染完，这个时候如果用户下拉刷新，就会导致组件内的 `index` 出问题。

**所以这里我把 `Deprecated` 代码提供出来了，起码让各位读者了解我在封装这个组件时候的思路，但是不推荐使用。**

## 🙄 版本记录

### 🚀 Version 3.0

🚀 Publish `Waterfall` & `WaterfallWithDimensions`。

### 🚀 Version 2.0.0

** 新版本组件 `命名方式` 和 `参数` 与 `FlatList` 一模一样。需要动态计算高度的，请自行安装 `1.x` 的最后一个版本 `1.9.0`。**

新版组件主要考虑到了 `业务场景` 和 `用户体验` 方面，只采用 `从左到右` 依次渲染。

- 业务场景:
  实际上数据源 `data` 都是运营的同学负责添加的，现在的 `APP` 基本上滑动不到底部，因为数据量实在很大。所以就算是几个列的高度差别很大，基本也不影响。这也是我决定最后一次改版的主要原因。

- 用户体验:
  1. 之前的版本都是动态计算，那就必然得等前一个渲染完了，才能渲染下一个。还是之前说的问题，会像 鸡 `🐔` 下蛋 `🥚` 一样，一个个的往外蹦。
  2. 新版本我加入了动画，交互效果看起来能好一些。

### 🍀 Version 1.0.0

🍀 Published react-native-staggered-list，支持分页加载 & Header & Footer 等功能。

- Version 1.0.1
  - 🗑 删除多余依赖。
  - ✍🏻 重命名 `StaggeredListView` → `StaggeredList`。
  - 🛠 更新 README.md。
- Version 1.1.0

  - 🆕 新增原生滑动事件的回调: `onScroll: (NativeSyntheticEvent<NativeScrollEvent>) => void`。

  - 🆕 新增 Header & Columns & Footer 测量高度的回调。

  有了以上这两个事件，就可以在使用的时候，实现 `TabBar` 的渐变以及吸顶效果。

- Version 1.1.1
  - 🐞 修改初始化 `measureResult`，防止 `header` 或者 `footer` 为 `null` 造成的回调参数为空的 BUG。
- Version 1.2.0
  - 🆕 新增下拉刷新功能 `onrefresh: () => void`。
  - 🛠 更新 README.md，添加运行截图，以及示例代码。
- Version 1.2.1
  - 🛠 修改 README.md。
- Version 1.3.0
  - 🆕 新增 `Columns` 样式自定义，可以自己调节 `Header` 和 `Columns` 之间的距离，也可以自己调节 `Columns` 和屏幕两边的边距。
- Version 1.4.0
  - 🗑 移除原来除了测量除了 `header` 和 `footer` 测量的逻辑，直接从左到右每一列挨个填充 `Item`。
- Version 1.4.1
  - 🐞 潜藏的 BUG。
- Version 1.4.2
  - 🐞 瀑布流渲染的错误。
- Version 1.5.0
  - 🚀 综合 `从左到右依次填充` 和 `最小高度填充` 两种方式，使瀑布流两边高度尽量一致。
  - 🆕 新增 `List` → `Item` 右下角的 `index`，便于直观的看到渲染顺序和效果。
- Version 1.5.1
  - 🐞。
- Version 1.6.0

  - 🚀 全新升级: 最外层由 `ScrollView` → `VirtualizedList`，包括内层的 `View` 堆砌也换成了 `VirtualizedList`。而且还解决了一些奇怪的问题，比如之前遇见过把 `Banner` 放到 `Header` 里面无法自动轮播，必须要手动碰一下才可以。

- Version 1.6.1
  - 🗑 删除 `Header` 以及 `Footer` 的测量的回调。
- Version 1.6.2
  - 🐞 `RefreshControl` 报错。
- Version 1.7.0

  - 🐞 新增下拉刷新的防抖的处理，防止用户不断下拉刷新造成重复渲染的 BUG。
  - 💄 优化瀑布流排列，不可见区域采用延时处理，排列更为准确。

- Version 1.7.1
  - 🐞 修改了一下防抖的时机，改为 `onRefresh()` 回调前就进行处理。
- Version 1.7.2
  - 🐞 还是防抖的逻辑，不要控制 `refreshing`，控制 `r` → `setR(Math.random())`。
- Version 1.7.3
  - 🛠 更新 README.md。
- Version 1.8.0
  - 🆕 新增泛型 `ItemT` 的支持。
- Version 1.8.1
  - 🛠 修改 README.md。
- Version 1.9.0

  - 🐞 修改加载完成 `onLoadComplete()` 的逻辑，因为当数据量比较大的时候，即使你不滑动，他也会不断 `onLoadComplete()`。

    - 1. 资源的浪费，不断加载下一页，会导致服务端的压力也变大。
    - 2. 下拉刷新有 `BUG`，为了让每一列能得到比较准确的高度，我会在添加的时候，加一个 `计时器`，如果他在不断的渲染的过程中，你突然下拉刷新，状态不好控制，会有意想不到的 `BUG` 出现。

所以这次更新，我回调的逻辑是 `每一列都滑动到底部了`，并且 `数据渲染完了`，这个时候我再去回调。


```javascript
 <Waterfall
           style={{width:'100%'}}
                            data={[1,2,3,4,5,6]}
                            renderItem={(item,index) => <View style={{width:170,height:index%2==0?90:120,backgroundColor:index%2==0?"#ccc":"#d8d8d8",marginTop:5}}><Font>fdsfds{item}</Font></View>}
                            numColumns={2}
               />
```
