# [从一个bug 深入剖析 多节点情况时的React diff算法](https://github.com/void-syh/blog/issues/3)

### 从一个BUG说起
不多废话，直接上代码
```
const renderList = (value: number) => {
        return (
            <>
                <div key={value}>1</div>
                <div key={value}>2</div>
                <div key={value}>3</div>
                <div key={value}>4</div>
            </>
        );
    };
    return (
        <div>
            <Button
                onClick={() => {
                    setValue(0);
                }}
            >
                A
            </Button>
            <Button
                onClick={() => {
                    setValue(100);
                }}
            >
                B
            </Button>
            {renderList(value)}
        </div>
    );
```
只是一个很简单的场景，按钮会导致下面的列表来回切换，为了保证每次切换都会重新渲染，我们给每个组件（这里用div模拟一下）加上了key值，每次切换key都会变化，达到重新渲染的目的，于是乎，奇怪的bug诞生了。

为了方便看出效果，我们将显示改为了1234，列表最初展示为这样
![image](https://user-images.githubusercontent.com/50072042/218082892-a15c5fba-5fcd-4986-974c-737aebd5dbe5.png)
而在点击一次之后，就变成了这样
![image](https://user-images.githubusercontent.com/50072042/218083054-b547b9dd-c7ee-4081-a859-961a08bfb29a.png)
再来回切换时，下面本应只有四个组件的列表越切越多，每次切换都会增加，可是明明renderList只渲染了四个div，怎么会切换时多渲染呢？

其实从报错很容易看出，错误原因是在同一列表中我们使用了相同的key值，react diff进行了一些奇怪的操作，导致了bug的发生，那么带着这个问题，我们来看下react 多节点diff的时候都干了些啥。
### 调试BUG产生的原因
首先进入diff的入口，我们直接跳到对比list节点的地方
![image2022-8-9_14-14-41](https://user-images.githubusercontent.com/50072042/218083691-ffa2ec77-0abf-46de-802c-e1864b2aa7df.png)
可以看到此时新的要渲染的列表以一个数组的形式存储了下来
![image](https://user-images.githubusercontent.com/50072042/218083721-578177bb-3c3e-4efa-a937-c9d20b2d5c93.png)
最终进入了reconcileChildrenArray，也就是多节点的一个diff中去。
![image](https://user-images.githubusercontent.com/50072042/218083781-cb866dba-01f2-469a-929d-e10d816d6e1a.png)

第一步是校验一下key，当存在key校验失败时，会在console中进行提示，可以看到这里列表中都是相同的key，就报错了。
![image](https://user-images.githubusercontent.com/50072042/218083799-a73be027-573e-4d5f-b87d-6f76c205e220.png)

然后定义了一些前置的变量，这里我们先忽略这些变量到底是干啥的。
![image](https://user-images.githubusercontent.com/50072042/218083821-08af8ab1-ac0a-4f8b-9df6-72d643a139bf.png)

接下来我们进入了第一个循环，循环条件是old不为空，或者new的idx小于总长度时。
这里我们先不看全部情况，专注这个问题看下走了个什么流程。
![image](https://user-images.githubusercontent.com/50072042/218083970-863c8b0c-40bb-4ce1-a490-b90b25a04d92.png)

第一个if时，进入了else，将nextOld指向了old的下一个节点，也就是2这个节点。从这里我们不难发现，老节点实际上采用的是链表的形式，而新节点，采用的是数组的形式，来进行一个对比，所以我们最初需要将下一个老节点保存下来。
![image](https://user-images.githubusercontent.com/50072042/218083970-863c8b0c-40bb-4ce1-a490-b90b25a04d92.png)

接下来，新的节点进行了一个updateSlot操作，这里我们先不具体看updateSlot的代码，实际上这个updateSlot就是比对key值的来返回的，由于key值并不相同，所以返回了null，而在下一个if中，由于新节点返回了null，第一次循环就被终止了。
![image](https://user-images.githubusercontent.com/50072042/218084009-bc0e04cc-2a9e-4776-aca9-2cbe7cc2eb74.png)

而后的两个判断很明显我们都没有进入。
![image](https://user-images.githubusercontent.com/50072042/218084029-a1cb912d-9bea-4bc3-9da4-4cbbc11d8f11.png)

之后，非常关键的一步来了，因为old是个链表，为了方便查找，react将剩余的old节点通过mapRemainingChildren方法转换为了一个map。
![image](https://user-images.githubusercontent.com/50072042/218084043-c1df71c3-b49c-41e1-949c-7dc57f11b44e.png)

但是可以看到，在转换的过程中，如果有key值存在的话，他是会根据key值来存储的，而由于列表里四个div的key值都相同，再循环的过程中，最终最后一个将前面的全部覆盖了，只保留了最后一个4节点
![image](https://user-images.githubusercontent.com/50072042/218084087-f3ba2581-3b03-404f-9c07-aa58616cce85.png)

剩下的操作，新节点循环添加到fiber中，然后删除了oldMap中4这个节点，最后渲染。
![image](https://user-images.githubusercontent.com/50072042/218084106-97068027-1741-44e9-901b-ed0d9ca66a4f.png)

导致其实123节点根本就没有参与剩下的diff，最终也也没有被删除，最后的效果看起来像是增加了一样。
### 分析多节点diff的源码
接下来，我们全方位看看多节点diff源码，看看多节点到底是怎么diff的

首先进入入口函数reconcileChildrenArray，我们先来看看整体结构和前置工作
```
function reconcileChildrenArray(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChildren: Array<*>,
    lanes: Lanes,
  ): Fiber | null {
 
    // 通过循环变量key，校验key正确性，否则控制台提示
    if (__DEV__) {
      let knownKeys = null;
      for (let i = 0; i < newChildren.length; i++) {
        const child = newChildren[i];
        knownKeys = warnOnInvalidKey(child, knownKeys, returnFiber);
      }
    }
 
    // 最终diff完成后返回的fiber
    let resultingFirstChild: Fiber | null = null;
    // 用于保存中间状态
    let previousNewFiber: Fiber | null = null;
 
    // 旧的fiber
    let oldFiber = currentFirstChild;
    let lastPlacedIndex = 0;
    // 新fiber的索引值
    let newIdx = 0;
    // 下一个oldFiber（old存储为链表）
    let nextOldFiber = null;
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
        //第一次循环
        ...
    }
 
    if (newIdx === newChildren.length) {
        // 当新的索引值等与长度，即新节点已经遍历完了，进行一些操作后返回
        ...
        return resultingFirstChild;
    }
 
    if (oldFiber === null) {   
        // 当老节点等于null，即老节点遍历完了，进行一些操作后返回
         ...
        return resultingFirstChild;
    }
 
    // 新老节点都没遍历完，将old的链表结构转化为map结构
    const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
 
    for (; newIdx < newChildren.length; newIdx++) {
        // 第二次循环
        ...
    }
 
    if (shouldTrackSideEffects) {
        // 通过遍历，将old节点map中的节点全部删掉
        existingChildren.forEach(child => deleteChild(returnFiber, child));
    }
 
    ...
    return resultingFirstChild;
  }
```
首先我们来看看第一次循环
```
for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
      if (oldFiber.index > newIdx) {
        nextOldFiber = oldFiber;
        oldFiber = null;
      } else {
        // 通过链式结构取得下一个old节点
        nextOldFiber = oldFiber.sibling;
      }，
 
      const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        lanes,
      );
      // 说明新节点不能直接复用老节点，退出循环
      if (newFiber === null) {
        if (oldFiber === null) {
          oldFiber = nextOldFiber;
        }
        break;
      }
      if (shouldTrackSideEffects) {
        // 如果老节点存在，且新节点是替换老节点的了，就把老节点删掉
        if (oldFiber && newFiber.alternate === null) {
          deleteChild(returnFiber, oldFiber);
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
 
      // 新节点生成链式结构
      if (previousNewFiber === null) {
        // result挂上头结点
        resultingFirstChild = newFiber;
      } else {
        // 如果已经有头节点了，就往下链
        previousNewFiber.sibling = newFiber;
      }
      // previousNewFiber 变成自己链的下一个，以此生成链
      previousNewFiber = newFiber;
      oldFiber = nextOldFiber;
    }
```
```
function updateSlot(
    returnFiber: Fiber,
    oldFiber: Fiber | null,
    newChild: any,
    lanes: Lanes,
  ): Fiber | null {
    const key = oldFiber !== null ? oldFiber.key : null;
 
    // 文本节点
    if (
      (typeof newChild === 'string' && newChild !== '') ||
      typeof newChild === 'number'
    ) {
      // 文本节点不会有key值，如果老节点是个带key值的，不匹配，不能复用，返回null
      if (key !== null) {
        return null;
      }
      // 否则可以复用
      return updateTextNode(returnFiber, oldFiber, '' + newChild, lanes);
    }
 
    // object类型的节点
    if (typeof newChild === 'object' && newChild !== null) {
      switch (newChild.$$typeof) {
        case REACT_ELEMENT_TYPE: {
          if (newChild.key === key) {
            // key值相等，可以复用
            return updateElement(returnFiber, oldFiber, newChild, lanes);
          } else {
            // key值不相等，返回null
            return null;
          }
        }
        case REACT_PORTAL_TYPE: {
          if (newChild.key === key) {
            return updatePortal(returnFiber, oldFiber, newChild, lanes);
          } else {
            return null;
          }
        }
        case REACT_LAZY_TYPE: {
          const payload = newChild._payload;
          const init = newChild._init;
          return updateSlot(returnFiber, oldFiber, init(payload), lanes);
        }
      }
 
      if (isArray(newChild) || getIteratorFn(newChild)) {
        // 多节点 如果老节点带key值说明是单节点，返回 null
        if (key !== null) {
          return null;
        }
        return updateFragment(returnFiber, oldFiber, newChild, lanes, null);
      }
 
      throwOnInvalidObjectType(returnFiber, newChild);
    }
 
    // 校验报错
    if (__DEV__) {
      if (typeof newChild === 'function') {
        warnOnFunctionType(returnFiber);
      }
    }
 
    return null;
  }
```
第一次遍历完后新节点idx等于长度
```
if (newIdx === newChildren.length) {
      // 新节点遍历完成，说明新节点全部都可以复用老节点，将剩余的老节点全部删掉即可
      deleteRemainingChildren(returnFiber, oldFiber);
      ...
      return resultingFirstChild;
    }
```
第一次遍历完后旧节点为null
```
if (oldFiber === null) {
      // 老节点遍历完成，说明老节点已经全部复用完了，剩下的新节点直接创建
      for (; newIdx < newChildren.length; newIdx++) {
        //创建节点
        const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
        if (newFiber === null) {
          continue;
        }
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        // 链表连起来
        if (previousNewFiber === null) {
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
      ...
      return resultingFirstChild;
    }
```
生成旧节点的Map
```
const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
 
function mapRemainingChildren(
    returnFiber: Fiber,
    currentFirstChild: Fiber,
  ): Map<string | number, Fiber> {
    // 将剩余的未复用的旧节点生成map结构，方便查找
    const existingChildren: Map<string | number, Fiber> = new Map();
 
    let existingChild = currentFirstChild;
    // 存储都是用set结构，所以同key值的会被覆盖
    while (existingChild !== null) {
      if (existingChild.key !== null) {
        // 有key存key
        existingChildren.set(existingChild.key, existingChild);
      } else {
        //无key存index
        existingChildren.set(existingChild.index, existingChild);
      }
      existingChild = existingChild.sibling;
    }
    return existingChildren;
  }
```
第二次遍历
```
for (; newIdx < newChildren.length; newIdx++) {
  // 从map中找，看看有没有能复用的
  const newFiber = updateFromMap(
    existingChildren,
    returnFiber,
    newIdx,
    newChildren[newIdx],
    lanes,
  );
  // 如果从map里复用了，就要把map里已经复用的旧节点给删了
  if (newFiber !== null) {
    if (shouldTrackSideEffects) {
      if (newFiber.alternate !== null) {
        existingChildren.delete(
          newFiber.key === null ? newIdx : newFiber.key,
        );
      }
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    // 链表连起来
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
  }
}
```
```
function placeChild(
    newFiber: Fiber,
    lastPlacedIndex: number,
    newIndex: number,
  ): number {
    newFiber.index = newIndex;
    ...
    const current = newFiber.alternate;
    if (current !== null) {
      const oldIndex = current.index;
      if (oldIndex < lastPlacedIndex) {
        // 移动
        newFiber.flags |= Placement;
        return lastPlacedIndex;
      } else {
        // 不动
        return oldIndex;
      }
    } else {
      // 插入
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    }
  }
```
### 总结多节点diff的大致流程
整个多节点diff的大致流程如下：

- 新老遍历对比，能复用老节点就复用老节点（例如只在末尾添加，或只是删除了一个末尾的节点时，前面所有的节点都会进入此循环直接复用掉）当遇到不能复用的节点，退出第一次循环。
- 如果第一次循环结束后，新节点遍历完了说明新节点都可以复用老节点，而老节点还没遍历完，说明剩下的老节点都是不要的，直接删掉。
- 如果老节点遍历完了，说明老节点已经都被复用了，剩下的没遍历完的新节点直接创建即可。
- 新老都没遍历完，说明节点的位置被改变，或者有节点不能直接复用等，为了方便后续操作，将节点的链表根据key或index转换为Map。
- 遍历新节点，直接去老节点的Map中找，找到了就复用，并将老节点从map中删掉，找不到就创建新的
- 最后删除所有Map中剩余的老节点