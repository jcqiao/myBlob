# createElement创建vnode

``` javascript
// core/vdom/create-element.js
  export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```

## 看参数

1. content: vm
2. tag: div...(可能是内置:div/component)
3. data: vnodedata(vnode 一些属性 key/keep-alive) /flow/vnode.js
4. children: 子vnode(文本/vnode：for循环)
5. normalizationType: 用来 **规范** children
6. alwaysNormalize: 有没有render函数

- render函数中调用createELement生成vnode 有两个版本
- 无render函数 vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
- 有render函数 vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
- 区别在于最后一个参数

## _createElement

- createElement包装了_createElement实际上这里才是返回vnode真正方法

``` javascript
  export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  ...
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
```

- __ob__只要有这个属性的就是响应式数据，vnode不能是响应式数据，因此这里报错并返回了一个空的vnode
- normalizationType用来决定生成什么样的children -- **规范children**
- 为什么要规范？
  - 由于 Virtual DOM 实际上是一个树状结构，每一个 VNode 可能会有若干个子节点，这些子节点应该也是 VNode 的类型。_createElement 接收的第 4 个参数 children 是任意类型的，因此我们需要把它们规范成 VNode 类型。
  - src/core/vdom/helpers/normalzie-children.js

  1. SIMPLE_NORMALIZE === 1 是编译成render函数调用
  正常情况下编译生成的render函数的children都应该是vnode，但有一个例外，就是function component children生成的是数组，所以要遍历成一维

``` javascript
  export function simpleNormalizeChildren (children: any) {
  for (let i = 0; i < children.length; i++) {
    if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
    }
  }
  return children
}
```

  1. ALWAYS_NORMALIZE === 2。 稍微复杂些

``` javascript
export function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}
```

- 应用场景：

1. 只有一个节点是基本类型：创建textnode
2. children是数组：编译v-for/slot children为嵌套数组。调用normalizeArrayChildren处理children返回vnode数组
- normalizeArrayChildren
  - 若数组元素c(节点)是基础类型：返回textnode
  - 若就是vnode: 直接返回
  - 数组：递归调用normalizeArrayChildren

``` javascript
  function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
  const res = []
  let i, c, lastIndex, last
  for (i = 0; i < children.length; i++) {
    c = children[i]
    if (isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    //  nested
    if (Array.isArray(c)) {
      if (c.length > 0) {
        c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
        // merge adjacent text nodes
        if (isTextNode(c[0]) && isTextNode(last)) {
          res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
          c.shift()
        }
        res.push.apply(res, c)
      }
    } else if (isPrimitive(c)) {
      if (isTextNode(last)) {
        // merge adjacent text nodes
        // this is necessary for SSR hydration because text nodes are
        // essentially merged when rendered to HTML strings
        res[lastIndex] = createTextVNode(last.text + c)
      } else if (c !== '') {
        // convert primitive to vnode
        res.push(createTextVNode(c))
      }
    } else {
      if (isTextNode(c) && isTextNode(last)) {
        // merge adjacent text nodes
        res[lastIndex] = createTextVNode(last.text + c.text)
      } else {
        // default key for nested array children (likely generated by v-for)
        if (isTrue(children._isVList) &&
          isDef(c.tag) &&
          isUndef(c.key) &&
          isDef(nestedIndex)) {
          c.key = `__vlist${nestedIndex}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```

## 回到创建vnode

- 回到createElement函数中 规范化children后开始创建vnode
- 判断tag:string?y 内置节点？y 创建一个普通的vnode(div) :n compontents? 创建组件类型的vnode :或者 创建一个不知名的vnode(mydiv自定义节点)

``` javascript
  export function createElement (){
    let vnode, ns
    if (typeof tag === 'string') {
      let Ctor
      ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
      if (config.isReservedTag(tag)) {
        // platform built-in elements
        vnode = new VNode(
          config.parsePlatformTagName(tag), data, children,
          undefined, undefined, context
        )
      } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
        // component
        vnode = createComponent(Ctor, data, context, children, tag)
      } else {
        // unknown or unlisted namespaced elements
        // check at runtime because it may get assigned a namespace when its
        // parent normalizes children
        vnode = new VNode(
          tag, data, children,
          undefined, undefined, context
        )
      }
    } else {
      // direct component options / constructor
      vnode = createComponent(tag, data, context, children)
    }
  }
```

## 总结

- 我们大致了解了createElement，就是使用规范化的children来生成vnode。

### 捋下步骤

1. Vue.prototype._render调用vm.$createElement返回vnode 其实也是对其一个封装 vnode = render.call(vm._renderProxy, vm.$createElement)
2. $createElement 定义在initRender中vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true), createElement定义在core/vdom/create-element.js
3. 回到 mountComponent 函数的过程，我们已经知道 vm._render 是如何创建了一个 VNode，接下来就是要把这个 VNode 渲染成一个真实的 DOM 并渲染出来，这个过程是通过 vm._update 完成的，接下来分析一下这个过程。