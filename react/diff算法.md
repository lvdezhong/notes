# diff算法的作用

计算出Virtual DOM中真正变化的部分，并只针对该部分进行原生DOM操作，而非重新渲染整个页面。



# 传统diff算法

通过循环递归对节点进行依次对比，算法复杂度达到 O(n^3) ，n是树的节点数，这个有多可怕呢？——如果要展示1000个节点，得执行上亿次比较。。即便是CPU快能执行30亿条命令，也很难在一秒内计算出差异。



# React的diff算法

1. 什么是调和

将Virtual DOM树转换成actual DOM树的**最少操作的过程** 称为 调和 。

2. 什么是React diff算法？

diff算法是**调和的具体实现。**



# diff策略

React用 三大策略 将O(n^3)复杂度 转化为 **O(n)复杂度**

策略一（**tree diff**）：
 Web UI中DOM节点跨层级的移动操作特别少，可以忽略不计。

策略二（**component diff**）：
 拥有相同类的两个组件 生成相似的树形结构，
 拥有不同类的两个组件 生成不同的树形结构。

策略三（**element diff**）：
 对于同一层级的一组子节点，通过唯一id区分。



### tree diff

1. React通过updateDepth对Virtual DOM树进行**层级控制**。
2. 对树分层比较，两棵树 只对**同一层次节点** 进行比较。如果该节点不存在时，则该节点及其子节点会被完全删除，不会再进一步比较。
3. 只需遍历一次，就能完成整棵DOM树的比较。

![](https://raw.githubusercontent.com/lvdezhong/figure-bed/master/img/0c08dbb6b1e0745780de4d208ad51d34_r.jpeg)

**那么问题来了，如果DOM节点出现了跨层级操作,diff会咋办呢？**

如下图，A 节点（包括其子节点）整个被移动到 D 节点下，由于 React 只会简单的考虑同层级节点的位置变换，而对于不同层级的节点，只有创建和删除操作。当根节点发现子节点中 A 消失了，就会直接销毁 A；当 D 发现多了一个子节点 A，则会创建新的 A（包括子节点）作为其子节点。此时，React diff 的执行情况：**create A -> create B -> create C -> delete A**。

由此可发现，当出现节点跨层级移动时，并不会出现想象中的移动操作，而是以 A 为根节点的树被整个重新创建，这是一种影响 React 性能的操作，因此 **React 官方建议不要进行 DOM 节点跨层级的操作**

> 注意：在开发组件时，保持稳定的 DOM 结构会有助于性能的提升。例如，可以通过 CSS 隐藏或显示节点，而不是真的移除或添加 DOM 节点。

![](https://raw.githubusercontent.com/lvdezhong/figure-bed/master/img/d712a73769688afe1ef1a055391d99ed_r.png)

---

### component diff

React对不同的组件间的比较，有三种策略

1. 同一类型的两个组件，按原策略（层级比较）继续比较Virtual DOM树即可。
2. 同一类型的两个组件，组件A变化为组件B时，可能Virtual DOM没有任何变化，如果知道这点（变换的过程中，Virtual DOM没有改变），可节省大量计算时间，所以用户可以通过 **shouldComponentUpdate()** 来判断是否需要 判断计算。
3. 不同类型的组件，将一个（将被改变的）组件判断为dirty component（脏组件），从而**替换 整个组件的所有节点**。

如下图，当 component D 改变为 component G 时，即使这两个 component 结构相似，一旦 React 判断 D 和 G 是不同类型的组件，就不会比较二者的结构，而是直接删除 component D，重新创建 component G 以及其子节点。虽然当两个 component 是不同类型但结构相似时，React diff 会影响性能，但正如 React 官方博客所言：不同类型的 component 是很少存在相似 DOM tree 的机会，因此这种极端因素很难在实现开发过程中造成重大影响的。

![](https://raw.githubusercontent.com/lvdezhong/figure-bed/master/img/52654992aba15fc90e2dac8b2387d0c4_r.png)

---

### element diff

当节点处于同一层级时，React diff 提供了三种节点操作，分别为：**INSERT_MARKUP**（插入）、**MOVE_EXISTING**（移动）和 **REMOVE_NODE**（删除）。

* **INSERT_MARKUP**，新的 component 类型不在老集合里， 即是全新的节点，需要对新节点执行插入操作。
* **MOVE_EXISTING**，在老集合有新 component 类型，且 element 是可更新的类型，generateComponentChildren 已调用 receiveComponent，这种情况下 prevChild=nextChild，就需要做移动操作，可以复用以前的 DOM 节点。
* **REMOVE_NODE**，老 component 类型，在新集合里也有，但对应的 element 不同则不能直接复用和更新，需要执行删除操作，或者老 component 不在新集合里的，也需要执行删除操作。

如下图，老集合中包含节点：A、B、C、D，更新后的新集合中包含节点：B、A、D、C，此时新老集合进行 diff 差异化对比，发现 B != A，则创建并插入 B 至新集合，删除老集合 A；以此类推，创建并插入 A、D 和 C，删除 B、C 和 D。

![](https://raw.githubusercontent.com/lvdezhong/figure-bed/master/img/7541670c089b84c59b84e9438e92a8e9_r.png)



React 发现这类操作繁琐冗余，因为这些都是相同的节点，但由于位置发生变化，导致需要进行繁杂低效的删除、创建操作，其实只要对这些节点进行位置移动即可。

针对这一现象，React 提出优化策略：允许开发者对同一层级的同组子节点，添加唯一 key 进行区分，虽然只是小小的改动，性能上却发生了翻天覆地的变化！

**情形一：新旧集合中存在相同节点但位置不同时，如何移动节点**

![](https://raw.githubusercontent.com/lvdezhong/figure-bed/master/img/c0aa97d996de5e7f1069e97ca3accfeb_r.jpeg)

1. 看着上图的 B，React先从新中取得B，然后判断旧中是否存在相同节点B，当发现存在节点B后，就去判断是否移动B。
    B在旧 中的index=1，它的lastIndex=0，**不满足 index < lastIndex 的条件，因此 B 不做移动操作。此时，一个操作是，lastIndex=(index,lastIndex)中的较大数=1.**

   **注意：lastIndex有点像浮标，或者说是一个map的索引，一开始默认值是0，它会与map中的元素进行比较，比较完后，会改变自己的值的（取index和lastIndex的较大数）。**

2. 看着 A，A在旧的index=0，此时的lastIndex=1（因为先前与新的B比较过了），**满足index<lastIndex**，因此，对A进行移动操作，此时**lastIndex=max(index,lastIndex)=1**。

3. 看着D，同（1），不移动，由于D在旧的index=3，比较时，lastIndex=1，所以改变lastIndex=max(index,lastIndex)=3

4. 看着C，同（2），移动，C在旧的index=2，满足index<lastIndex（lastIndex=3），所以移动。

   **由于C已经是最后一个节点，所以diff操作结束。**

**情形二：新集合中有新加入的节点，旧集合中有删除的节点**

![](https://raw.githubusercontent.com/lvdezhong/figure-bed/master/img/7b9beae0cf0a5bc8c2e82d00c43d1c90_r.jpeg)

1. B不移动，不赘述，更新l astIndex=1
2. 新集合取得 E，发现旧不存在，故**在lastIndex=1的位置 创建E**，更新lastIndex=1
3. 新集合取得C，C不移动，更新lastIndex=2
4. 新集合取得A，A移动，同上，更新lastIndex=2
5. **新集合对比后，再对旧集合遍历。判断 新集合 没有，但 旧集合 有的元素（如D，新集合没有，旧集合有）**，发现 D，删除D，diff操作结束。

**diff的不足与待优化的地方**

![](https://raw.githubusercontent.com/lvdezhong/figure-bed/master/img/1b8dac5b9b3e4452dec8d5447d7717ad_r.jpeg)

看图的 D，此时D不移动，但它的index是最大的，导致更新lastIndex=3，从而使得其他元素A,B,C的index<lastIndex，导致A,B,C都要去移动。理想情况是只移动D，不移动A,B,C。

> 建议：在开发过程中，尽量减少类似将最后一个节点移动到列表首部的操作，当节点数量过大或更新操作过于频繁时，在一定程度上会影响 React 的渲染性能。

