#RN开发问题合集

## 1.RN StyleSheet默认样式
```css
{
    box-sizing: border-box;
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    flex-shrink: 0;
    border: 0 solid black;
    margin: 0;
    padding: 0
}
```

## 2.FlatList中keyExtractor

此函数用于为给定的 item 生成一个不重复的 key。Key 的作用是使 React 能够区分同类元素的不同个体，以便在刷新时能够确定其变化的位置，减少重新渲染的开销。若不指定此函数，则默认抽取item.key作为 key 值。若item.key也不存在，则使用数组下标。若需要修改item中的某字段时并需要更新视图，则需要修改data的引用
## 3.react native阻止事件冒泡

在View的属性中加上 onStartShouldSetResponderCapture={(ev) => true}；这样可以阻止该节点的子节点全部不能冒泡
