本篇文章是官方文档的翻译，英文原文请访问[官网](https://reactjs.org/docs/implementation-notes.html)

这个章节是[stack reconciler](https://reactjs.org/docs/codebase-overview.html#stack-reconciler)的一些实现说明.

它的技术性很强并假定你能完全理解React的公开API，以及它是如何划分为核心、渲染器和协调器的。如果你对React代码不是很熟悉，请先阅读[代码概览](https://reactjs.org/docs/codebase-overview.html)。

它还假定你能够理解[React组件、实例和元素](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)的区别。


Stack reconciler 被用在React 15 以及更早的版本中， 它在源代码中的位置是[src/renderers/shared/stack/reconciler](https://github.com/facebook/react/tree/15-stable/src/renderers/shared/stack/reconciler).

### 视频：从零开始构建React

[Paul O'Shannessy](https://twitter.com/zpao)给出了一个关于[从零开始构建React](https://www.youtube.com/watch?v=_MAD4Oly9yg)的讨论，在很大程度上对本文档给予了启发。

本文档与上边的视频都是对实际代码库的简化，因此你可以通过熟悉两者来更好地理解。

### 概述

协调器本身没有公共 API. 但是诸如React DOM 和React Native的[渲染器](https://reactjs.org/docs/codebase-overview.html#stack-renderers)使用它依据用户所编写的React组件来有效地更新用户界面.

### 以递归过程的形式装载

让我们考虑首次装载组件的情形：

```js
ReactDOM.render(<App />, rootEl);
```

React DOM会将 `<App />`传递给协调器。请记住， `<App />`是一个React元素，也就是说是对哪些要渲染的东西的说明。你可以把它看成一个普通的对象：

```js
console.log(<App />);
// { type: App, props: {} }
```

协调器（reconciler）会检查 `App`是类还是函数。如果 `App` 是函数，协调器会调用`App(props)`来获取所渲染的元素。如果`App`是类，协调器则会使用`new App(props)`创建一个`App`实例，调用 `componentWillMount()` 生命周期方法，进而调用 `render()` 方法来获取所渲染的元素。无论如何，协调器都会学习App元素的“渲染行为”。

此过程是递归的。`App` 可能渲染为`<Greeting />`，而`<Greeting />`可能渲染为 `<Button />`，如此类推。因为协调器会依次学习他们各自将如何渲染，所以协调器会递归地“向下钻取”所有用户定义组件。

你可以通过如下伪代码来理解该过程：

```js
function isClass(type) {
  // React.Component的子类都会含有这一标志
  return (
    Boolean(type.prototype) &&
    Boolean(type.prototype.isReactComponent)
  );
}

// This function takes a React element (e.g. <App />)
// and returns a DOM or Native node representing the mounted tree.
// 此函数读取一个React元素（例如<App />）
// 并返回一个表达所装载树的DOM或内部节点。
function mount(element) {
  var type = element.type;
  var props = element.props;

  // 我们以此判断所渲染元素：
  // 是以函数型运行该类型
  // 还是创建新实例并调用render()。
  var renderedElement;
  if (isClass(type)) {
    // Component class
    var publicInstance = new type(props);
    // Set the props
    publicInstance.props = props;
    // Call the lifecycle if necessary
    if (publicInstance.componentWillMount) {
      publicInstance.componentWillMount();
    }
    // 调用render()以获取所渲染元素
    renderedElement = publicInstance.render();
  } else {
    // 组件函数
    renderedElement = type(props);
  }

  // 该过程是递归实现，原因在于组件可能返回一个其它组件类型的元素。
  return mount(renderedElement);

  // 注意：该实现不完整，且将无穷递归！ 它只处理<App />或<Button />等元素。尚不处理<div />或<p />等元素。
}

var rootEl = document.getElementById('root');
var node = mount(<App />);
rootEl.appendChild(node);
```

>**注意:**
>
>这的确是一段伪代码。它与真实的实现不同。它会导致栈溢出，因为我们还没有讨论何时停止递归。

让我们回顾一下上面示例中的几个关键概念:

* React元素是表示组件类型（例如 `App`）与属性的普通对象。
* 用户定义组件（例如 `App`）可以为类或者函数，但它们都会被“渲染为”元素。
* “装载”（Mounting）是一个递归过程，当给定顶级React元素（例如<App />）时创建DOM或内部节点树。

### 装载主机元素（Mounting Host Elements）

该过程将没有任何意义，如果最终没有渲染内容到屏幕上。

除了用户定义的（“复合”）组件外, React元素还可能表示特定于平台的（“主机”）组件。例如，`Button`可能会从其渲染方法中返回 `<div />` 。

如果元素的type属性是一个字符串，即表示我们正在处理一个主机元素（host element）：

```js
console.log(<div />);
// { type: 'div', props: {} }
```

主机元素（host elements）不存在关联的用户定义代码。

当协调器遇到主机元素（host element）时，它会让渲染器（renderer）装载它（mounting）。例如，React DOM将会创建一个DOM节点。

如果主机元素（host element）有子级，协调器（reconciler）则会用上述相同算法递归地将它们装载。而不管子级是主机元素（如`<div><hr /></div>`）还是混合元素（如`<div><Button /></div>`）或是两者兼有。

由子级组件生成的DOM节点将被追加到DOM父节点，同时整的DOM结构会被递归装配。

>**注意:**
>
>协调器本身(reconciler)并不与DOM捆绑。装载(mounting)的具体结果（有时在源代码中称为“装载映像”）取决于渲染器(renderer)，可能为 DOM节点（React DOM）、字符串（React DOM服务器）或表示本机视图的数值（React Native）。


我们来扩展一下代码，以处理主机元素（host elements）:

```js
function isClass(type) {
  //  React.Component 子类含有这一标志
  return (
    Boolean(type.prototype) &&
    Boolean(type.prototype.isReactComponent)
  );
}

// 该函数仅处理含复合类型的元素。  例如，它处理<App />和<Button />，但不处理<div />。
function mountComposite(element) {
  var type = element.type;
  var props = element.props;

  var renderedElement;
  if (isClass(type)) {
    // 组件类
    var publicInstance = new type(props);
    // 设置属性
    publicInstance.props = props;
    // 若必要，则调用生命周期函数
    if (publicInstance.componentWillMount) {
      publicInstance.componentWillMount();
    }
    renderedElement = publicInstance.render();
  } else if (typeof type === 'function') {
    // 组件函数
    renderedElement = type(props);
  }

  // 该过程是递归，一旦该元素为主机（如<div />｝而非复合（如<App />）时，则逐渐结束
  return mount(renderedElement);
}

// 该函数仅处理含主机类型的元素(handles elements with a host type)。 例如，它处理<div />和<p />但不处理<App />。
function mountHost(element) {
  var type = element.type;
  var props = element.props;
  var children = props.children || [];
  if (!Array.isArray(children)) {
    children = [children];
  }
  children = children.filter(Boolean);

  // 该代码块不可出现在协调器(reconciler)中。
  // 不同渲染器(renderers)可能会以不同方式初始化节点。
  // 例如，React Native会生成iOS或Android视图。
  var node = document.createElement(type);
  Object.keys(props).forEach(propName => {
    if (propName !== 'children') {
      node.setAttribute(propName, props[propName]);
    }
  });

  // 装载子节点
  children.forEach(childElement => {
    // 子节点有可能是主机元素（如<div />）或复合元素（如<Button />）.
    // 所以我们应该递归的装载
    var childNode = mount(childElement);

    // 此行代码仍是特定于渲染器的。不同的渲染器则会使用不同的方法
    node.appendChild(childNode);
  });

  // 返回DOM节点作为装载结果
  // 此处即为递归结束.
  return node;
}

function mount(element) {
  var type = element.type;
  if (typeof type === 'function') {
    // 用户定义的组件
    return mountComposite(element);
  } else if (typeof type === 'string') {
    // 平台相关的组件，比如说浏览器中的div，ios和安卓中的视图
    return mountHost(element);
  }
}

var rootEl = document.getElementById('root');
var node = mount(<App />);
rootEl.appendChild(node);
```

该代码能够工作但仍与协调器（reconciler）的真正实现相差甚远。其所缺少的关键部分是对更新的支持。

### 介绍内部实例

React 的关键特征是您可以重新渲染所有内容, 它不会重新创建 DOM 或重置状态:

```js
ReactDOM.render(<App />, rootEl);
// 应该重新使用现存的 DOM:
ReactDOM.render(<App />, rootEl);
```

但是, 上面的实现只知道如何装载初始树。它无法对其执行更新, 因为它没有存储所有必需的信息, 例如所有 `publicInstance` , 
或者哪个 DOM 节点 对应于哪些组件。

堆栈协调（stack reconciler）的基本代码是通过使 mount () 函数成为一个方法并将其放在类上来解决这一问题。
这种方式有一些缺陷，但是目前代码中仍然使用的是这种方式。不过目前我们也正在重写[协调器（reconciler）](https://reactjs.org/docs/codebase-overview.html#fiber-reconciler)

我们将创建两个类: DOMComponent 和 CompositeComponent , 而不是单独的 mountHost 和 mountComposite 函数。

两个类都有一个接受 element 的构造函数, 以及一个能返回已装入节点的 mount () 方法。我们将用一个能实例化正确类的工厂函数替换掉之前
例子里的mount函数：

```js
function instantiateComponent(element) {
  var type = element.type;
  if (typeof type === 'function') {
    // 用户自定义组件
    return new CompositeComponent(element);
  } else if (typeof type === 'string') {
    // 特定于平台的组件
    return new DOMComponent(element);
  }  
}
```

首先, 让我们考虑如何实现 `CompositeComponent`:

```js
class CompositeComponent {
  constructor(element) {
    this.currentElement = element;
    this.renderedComponent = null;
    this.publicInstance = null;
  }

  getPublicInstance() {
    // 针对复合组合, 返回类的实例.
    return this.publicInstance;
  }

  mount() {
    var element = this.currentElement;
    var type = element.type;
    var props = element.props;

    var publicInstance;
    var renderedElement;
    if (isClass(type)) {
      // 组件类
      publicInstance = new type(props);
      // 设置属性
      publicInstance.props = props;
      // 如果有必要，调用生命周期
      if (publicInstance.componentWillMount) {
        publicInstance.componentWillMount();
      }
      renderedElement = publicInstance.render();
    } else if (typeof type === 'function') {
      // Component function
      publicInstance = null;
      renderedElement = type(props);
    }

    // Save the public instance
    this.publicInstance = publicInstance;

    // 通过element实例化内部的child实例，这个实例有可能是DOMComponent，比如<div /> or <p />
    // 也可能是CompositeComponent 比如说<App /> or <Button />
    var renderedComponent = instantiateComponent(renderedElement);
    this.renderedComponent = renderedComponent;

    // 增加渲染输出
    return renderedComponent.mount();
  }
}
```

这与我们以前的 `mountComposite()` 实现没有太大的不同, 但现在我们可以保存一些信息, 
比如`this.currentElement`、`this.renderedComponent` 和 `this.publicInstance` ,这些保存的信息会在更新期间被使用。

请注意, `CompositeComponent`的实例与用户提供的 `element.type` 的实例不是一回事。
`CompositeComponent`是我们的协调器（reconciler）的一个实现细节, 从不向用户公开。
用户自定义类是我们从 `element.type` 读取的，并且通过 `CompositeComponent` 创建它的一个实例。

为避免混乱，我们将`CompositeComponent`和`DOMComponent`的实例称为“内部实例”。
由于它们的存在, 我们可以将一些长寿数据（ong-lived）与它们关联起来。只有渲染器（renderer）和协调器（reconciler）知道它们的存在。

另一方面, 我们将用户定义的类的实例称为 "公共实例"（public instance）。公共实例是您在 `render()` 和自定义组件的其他方法中看到的 `this`

`mountHost()` 函数被重构为 `DOMComponent` 类上的 `mount()`方法, 也看起来很熟悉:

```js
class DOMComponent {
  constructor(element) {
    this.currentElement = element;
    this.renderedChildren = [];
    this.node = null;
  }

  getPublicInstance() {
    // For DOM components, only expose the DOM node.
    return this.node;
  }

  mount() {
    var element = this.currentElement;
    var type = element.type;
    var props = element.props;
    var children = props.children || [];
    if (!Array.isArray(children)) {
      children = [children];
    }

    // Create and save the node
    var node = document.createElement(type);
    this.node = node;

    // Set the attributes
    Object.keys(props).forEach(propName => {
      if (propName !== 'children') {
        node.setAttribute(propName, props[propName]);
      }
    });

    // Create and save the contained children.
    // Each of them can be a DOMComponent or a CompositeComponent,
    // depending on whether the element type is a string or a function.
    var renderedChildren = children.map(instantiateComponent);
    this.renderedChildren = renderedChildren;

    // Collect DOM nodes they return on mount
    var childNodes = renderedChildren.map(child => child.mount());
    childNodes.forEach(childNode => node.appendChild(childNode));

    // Return the DOM node as mount result
    return node;
  }
}
```
从 mountHost () 重构后的主要区别在于, 我们现在将 `this.node`  和 `this.renderedChildren`  与内部 DOM 组件实例相关联。
我们还将使用它们在将来应用非破坏性更新。

As a result, each internal instance, composite or host, now points to its child internal instances. To help visualize this, if a functional `<App>` component renders a `<Button>` class component, and `Button` class renders a `<div>`, the internal instance tree would look like this:

```js
[object CompositeComponent] {
  currentElement: <App />,
  publicInstance: null,
  renderedComponent: [object CompositeComponent] {
    currentElement: <Button />,
    publicInstance: [object Button],
    renderedComponent: [object DOMComponent] {
      currentElement: <div />,
      node: [object HTMLDivElement],
      renderedChildren: []
    }
  }
}
```

In the DOM you would only see the `<div>`. However the internal instance tree contains both composite and host internal instances.

The composite internal instances need to store:

* The current element.
* The public instance if element type is a class.
* The single rendered internal instance. It can be either a `DOMComponent` or a `CompositeComponent`.

The host internal instances need to store:

* The current element.
* The DOM node.
* All the child internal instances. Each of them can be either a `DOMComponent` or a `CompositeComponent`.

If you're struggling to imagine how an internal instance tree is structured in more complex applications, [React DevTools](https://github.com/facebook/react-devtools) can give you a close approximation, as it highlights host instances with grey, and composite instances with purple:

 <img src="../images/docs/implementation-notes-tree.png" width="500" style="max-width: 100%" alt="React DevTools tree" />

To complete this refactoring, we will introduce a function that mounts a complete tree into a container node, just like `ReactDOM.render()`. It returns a public instance, also like `ReactDOM.render()`:

```js
function mountTree(element, containerNode) {
  // Create the top-level internal instance
  var rootComponent = instantiateComponent(element);

  // Mount the top-level component into the container
  var node = rootComponent.mount();
  containerNode.appendChild(node);

  // Return the public instance it provides
  var publicInstance = rootComponent.getPublicInstance();
  return publicInstance;
}

var rootEl = document.getElementById('root');
mountTree(<App />, rootEl);
```

### Unmounting

Now that we have internal instances that hold onto their children and the DOM nodes, we can implement unmounting. For a composite component, unmounting calls a lifecycle hook and recurses.

```js
class CompositeComponent {

  // ...

  unmount() {
    // Call the lifecycle hook if necessary
    var publicInstance = this.publicInstance;
    if (publicInstance) {
      if (publicInstance.componentWillUnmount) {
        publicInstance.componentWillUnmount();
      }
    }

    // Unmount the single rendered component
    var renderedComponent = this.renderedComponent;
    renderedComponent.unmount();
  }
}
```

For `DOMComponent`, unmounting tells each child to unmount:

```js
class DOMComponent {

  // ...

  unmount() {
    // Unmount all the children
    var renderedChildren = this.renderedChildren;
    renderedChildren.forEach(child => child.unmount());
  }
}
```

In practice, unmounting DOM components also removes the event listeners and clears some caches, but we will skip those details.

We can now add a new top-level function called `unmountTree(containerNode)` that is similar to `ReactDOM.unmountComponentAtNode()`:

```js
function unmountTree(containerNode) {
  // Read the internal instance from a DOM node:
  // (This doesn't work yet, we will need to change mountTree() to store it.)
  var node = containerNode.firstChild;
  var rootComponent = node._internalInstance;

  // Unmount the tree and clear the container
  rootComponent.unmount();
  containerNode.innerHTML = '';
}
```

In order for this to work, we need to read an internal root instance from a DOM node. We will modify `mountTree()` to add the `_internalInstance` property to the root DOM node. We will also teach `mountTree()` to destroy any existing tree so it can be called multiple times:

```js
function mountTree(element, containerNode) {
  // Destroy any existing tree
  if (containerNode.firstChild) {
    unmountTree(containerNode);
  }

  // Create the top-level internal instance
  var rootComponent = instantiateComponent(element);

  // Mount the top-level component into the container
  var node = rootComponent.mount();
  containerNode.appendChild(node);

  // Save a reference to the internal instance
  node._internalInstance = rootComponent;

  // Return the public instance it provides
  var publicInstance = rootComponent.getPublicInstance();
  return publicInstance;
}
```

Now, running `unmountTree()`, or running `mountTree()` repeatedly, removes the old tree and runs the `componentWillUnmount()` lifecycle hook on components.

### Updating

In the previous section, we implemented unmounting. However React wouldn't be very useful if each prop change unmounted and mounted the whole tree. The goal of the reconciler is to reuse existing instances where possible to preserve the DOM and the state:

```js
var rootEl = document.getElementById('root');

mountTree(<App />, rootEl);
// Should reuse the existing DOM:
mountTree(<App />, rootEl);
```

We will extend our internal instance contract with one more method. In addition to `mount()` and `unmount()`, both `DOMComponent` and `CompositeComponent` will implement a new method called `receive(nextElement)`:

```js
class CompositeComponent {
  // ...

  receive(nextElement) {
    // ...
  }
}

class DOMComponent {
  // ...

  receive(nextElement) {
    // ...
  }
}
```

Its job is to do whatever is necessary to bring the component (and any of its children) up to date with the description provided by the `nextElement`.

This is the part that is often described as "virtual DOM diffing" although what really happens is that we walk the internal tree recursively and let each internal instance receive an update.

### Updating Composite Components

When a composite component receives a new element, we run the `componentWillUpdate()` lifecycle hook.

Then we re-render the component with the new props, and get the next rendered element:

```js
class CompositeComponent {

  // ...

  receive(nextElement) {
    var prevProps = this.currentElement.props;
    var publicInstance = this.publicInstance;
    var prevRenderedComponent = this.renderedComponent;
    var prevRenderedElement = prevRenderedComponent.currentElement;

    // Update *own* element
    this.currentElement = nextElement;
    var type = nextElement.type;
    var nextProps = nextElement.props;

    // Figure out what the next render() output is
    var nextRenderedElement;
    if (isClass(type)) {
      // Component class
      // Call the lifecycle if necessary
      if (publicInstance.componentWillUpdate) {
        publicInstance.componentWillUpdate(nextProps);
      }
      // Update the props
      publicInstance.props = nextProps;
      // Re-render
      nextRenderedElement = publicInstance.render();
    } else if (typeof type === 'function') {
      // Component function
      nextRenderedElement = type(nextProps);
    }

    // ...
```

Next, we can look at the rendered element's `type`. If the `type` has not changed since the last render, the component below can also be updated in place.

For example, if it returned `<Button color="red" />` the first time, and `<Button color="blue" />` the second time, we can just tell the corresponding internal instance to `receive()` the next element:

```js
    // ...

    // If the rendered element type has not changed,
    // reuse the existing component instance and exit.
    if (prevRenderedElement.type === nextRenderedElement.type) {
      prevRenderedComponent.receive(nextRenderedElement);
      return;
    }

    // ...
```

However, if the next rendered element has a different `type` than the previously rendered element, we can't update the internal instance. A `<button>` can't "become" an `<input>`.

Instead, we have to unmount the existing internal instance and mount the new one corresponding to the rendered element type. For example, this is what happens when a component that previously rendered a `<button />` renders an `<input />`:

```js
    // ...

    // If we reached this point, we need to unmount the previously
    // mounted component, mount the new one, and swap their nodes.

    // Find the old node because it will need to be replaced
    var prevNode = prevRenderedComponent.getHostNode();

    // Unmount the old child and mount a new child
    prevRenderedComponent.unmount();
    var nextRenderedComponent = instantiateComponent(nextRenderedElement);
    var nextNode = nextRenderedComponent.mount();

    // Replace the reference to the child
    this.renderedComponent = nextRenderedComponent;

    // Replace the old node with the new one
    // Note: this is renderer-specific code and
    // ideally should live outside of CompositeComponent:
    prevNode.parentNode.replaceChild(nextNode, prevNode);
  }
}
```

To sum this up, when a composite component receives a new element, it may either delegate the update to its rendered internal instance, or unmount it and mount a new one in its place.

There is another condition under which a component will re-mount rather than receive an element, and that is when the element's `key` has changed. We don't discuss `key` handling in this document because it adds more complexity to an already complex tutorial.

Note that we needed to add a method called `getHostNode()` to the internal instance contract so that it's possible to locate the platform-specific node and replace it during the update. Its implementation is straightforward for both classes:

```js
class CompositeComponent {
  // ...

  getHostNode() {
    // Ask the rendered component to provide it.
    // This will recursively drill down any composites.
    return this.renderedComponent.getHostNode();
  }
}

class DOMComponent {
  // ...

  getHostNode() {
    return this.node;
  }  
}
```

### Updating Host Components

Host component implementations, such as `DOMComponent`, update differently. When they receive an element, they need to update the underlying platform-specific view. In case of React DOM, this means updating the DOM attributes:

```js
class DOMComponent {
  // ...

  receive(nextElement) {
    var node = this.node;
    var prevElement = this.currentElement;
    var prevProps = prevElement.props;
    var nextProps = nextElement.props;    
    this.currentElement = nextElement;

    // Remove old attributes.
    Object.keys(prevProps).forEach(propName => {
      if (propName !== 'children' && !nextProps.hasOwnProperty(propName)) {
        node.removeAttribute(propName);
      }
    });
    // Set next attributes.
    Object.keys(nextProps).forEach(propName => {
      if (propName !== 'children') {
        node.setAttribute(propName, nextProps[propName]);
      }
    });

    // ...
```

Then, host components need to update their children. Unlike composite components, they might contain more than a single child.

In this simplified example, we use an array of internal instances and iterate over it, either updating or replacing the internal instances depending on whether the received `type` matches their previous `type`. The real reconciler also takes element's `key` in the account and track moves in addition to insertions and deletions, but we will omit this logic.

We collect DOM operations on children in a list so we can execute them in batch:

```js
    // ...

    // These are arrays of React elements:
    var prevChildren = prevProps.children || [];
    if (!Array.isArray(prevChildren)) {
      prevChildren = [prevChildren];
    }
    var nextChildren = nextProps.children || [];
    if (!Array.isArray(nextChildren)) {
      nextChildren = [nextChildren];
    }
    // These are arrays of internal instances:
    var prevRenderedChildren = this.renderedChildren;
    var nextRenderedChildren = [];

    // As we iterate over children, we will add operations to the array.
    var operationQueue = [];

    // Note: the section below is extremely simplified!
    // It doesn't handle reorders, children with holes, or keys.
    // It only exists to illustrate the overall flow, not the specifics.

    for (var i = 0; i < nextChildren.length; i++) {
      // Try to get an existing internal instance for this child
      var prevChild = prevRenderedChildren[i];

      // If there is no internal instance under this index,
      // a child has been appended to the end. Create a new
      // internal instance, mount it, and use its node.
      if (!prevChild) {
        var nextChild = instantiateComponent(nextChildren[i]);
        var node = nextChild.mount();

        // Record that we need to append a node
        operationQueue.push({type: 'ADD', node});
        nextRenderedChildren.push(nextChild);
        continue;
      }

      // We can only update the instance if its element's type matches.
      // For example, <Button size="small" /> can be updated to
      // <Button size="large" /> but not to an <App />.
      var canUpdate = prevChildren[i].type === nextChildren[i].type;

      // If we can't update an existing instance, we have to unmount it
      // and mount a new one instead of it.
      if (!canUpdate) {
        var prevNode = prevChild.getHostNode();
        prevChild.unmount();

        var nextChild = instantiateComponent(nextChildren[i]);
        var nextNode = nextChild.mount();

        // Record that we need to swap the nodes
        operationQueue.push({type: 'REPLACE', prevNode, nextNode});
        nextRenderedChildren.push(nextChild);
        continue;
      }

      // If we can update an existing internal instance,
      // just let it receive the next element and handle its own update.
      prevChild.receive(nextChildren[i]);
      nextRenderedChildren.push(prevChild);
    }

    // Finally, unmount any children that don't exist:
    for (var j = nextChildren.length; j < prevChildren.length; j++) {
      var prevChild = prevRenderedChildren[j];
      var node = prevChild.getHostNode();
      prevChild.unmount();

      // Record that we need to remove the node
      operationQueue.push({type: 'REMOVE', node});
    }

    // Point the list of rendered children to the updated version.
    this.renderedChildren = nextRenderedChildren;

    // ...
```

As the last step, we execute the DOM operations. Again, the real reconciler code is more complex because it also handles moves:

```js
    // ...

    // Process the operation queue.
    while (operationQueue.length > 0) {
      var operation = operationQueue.shift();
      switch (operation.type) {
      case 'ADD':
        this.node.appendChild(operation.node);
        break;
      case 'REPLACE':
        this.node.replaceChild(operation.nextNode, operation.prevNode);
        break;
      case 'REMOVE':
        this.node.removeChild(operation.node);
        break;
      }
    }
  }
}
```

And that is it for updating host components.

### Top-Level Updates

Now that both `CompositeComponent` and `DOMComponent` implement the `receive(nextElement)` method, we can change the top-level `mountTree()` function to use it when the element `type` is the same as it was the last time:

```js
function mountTree(element, containerNode) {
  // Check for an existing tree
  if (containerNode.firstChild) {
    var prevNode = containerNode.firstChild;
    var prevRootComponent = prevNode._internalInstance;
    var prevElement = prevRootComponent.currentElement;

    // If we can, reuse the existing root component
    if (prevElement.type === element.type) {
      prevRootComponent.receive(element);
      return;
    }

    // Otherwise, unmount the existing tree
    unmountTree(containerNode);
  }

  // ...

}
```

Now calling `mountTree()` two times with the same type isn't destructive:

```js
var rootEl = document.getElementById('root');

mountTree(<App />, rootEl);
// Reuses the existing DOM:
mountTree(<App />, rootEl);
```

These are the basics of how React works internally.

### What We Left Out

This document is simplified compared to the real codebase. There are a few important aspects we didn't address:

* Components can render `null`, and the reconciler can handle "empty slots" in arrays and rendered output.

* The reconciler also reads `key` from the elements, and uses it to establish which internal instance corresponds to which element in an array. A bulk of complexity in the actual React implementation is related to that.

* In addition to composite and host internal instance classes, there are also classes for "text" and "empty" components. They represent text nodes and the "empty slots" you get by rendering `null`.

* Renderers use [injection](/docs/codebase-overview.html#dynamic-injection) to pass the host internal class to the reconciler. For example, React DOM tells the reconciler to use `ReactDOMComponent` as the host internal instance implementation.

* The logic for updating the list of children is extracted into a mixin called `ReactMultiChild` which is used by the host internal instance class implementations both in React DOM and React Native.

* The reconciler also implements support for `setState()` in composite components. Multiple updates inside event handlers get batched into a single update.

* The reconciler also takes care of attaching and detaching refs to composite components and host nodes.

* Lifecycle hooks that are called after the DOM is ready, such as `componentDidMount()` and `componentDidUpdate()`, get collected into "callback queues" and are executed in a single batch.

* React puts information about the current update into an internal object called "transaction". Transactions are useful for keeping track of the queue of pending lifecycle hooks, the current DOM nesting for the warnings, and anything else that is "global" to a specific update. Transactions also ensure React "cleans everything up" after updates. For example, the transaction class provided by React DOM restores the input selection after any update.

### Jumping into the Code

* [`ReactMount`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/dom/client/ReactMount.js) is where the code like `mountTree()` and `unmountTree()` from this tutorial lives. It takes care of mounting and unmounting top-level components. [`ReactNativeMount`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/native/ReactNativeMount.js) is its React Native analog.
* [`ReactDOMComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/dom/shared/ReactDOMComponent.js) is the equivalent of `DOMComponent` in this tutorial. It implements the host component class for React DOM renderer. [`ReactNativeBaseComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/native/ReactNativeBaseComponent.js) is its React Native analog.
* [`ReactCompositeComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js) is the equivalent of `CompositeComponent` in this tutorial. It handles calling user-defined components and maintaining their state.
* [`instantiateReactComponent`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/instantiateReactComponent.js) contains the switch that picks the right internal instance class to construct for an element. It is equivalent to `instantiateComponent()` in this tutorial.

* [`ReactReconciler`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactReconciler.js) is a wrapper with `mountComponent()`, `receiveComponent()`, and `unmountComponent()` methods. It calls the underlying implementations on the internal instances, but also includes some code around them that is shared by all internal instance implementations.

* [`ReactChildReconciler`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactChildReconciler.js) implements the logic for mounting, updating, and unmounting children according to the `key` of their elements.

* [`ReactMultiChild`](https://github.com/facebook/react/blob/83381c1673d14cd16cf747e34c945291e5518a86/src/renderers/shared/stack/reconciler/ReactMultiChild.js) implements processing the operation queue for child insertions, deletions, and moves independently of the renderer.

* `mount()`, `receive()`, and `unmount()` are really called `mountComponent()`, `receiveComponent()`, and `unmountComponent()` in React codebase for legacy reasons, but they receive elements.

* Properties on the internal instances start with an underscore, e.g. `_currentElement`. They are considered to be read-only public fields throughout the codebase.

### Future Directions

Stack reconciler has inherent limitations such as being synchronous and unable to interrupt the work or split it in chunks. There is a work in progress on the [new Fiber reconciler](/docs/codebase-overview.html#fiber-reconciler) with a [completely different architecture](https://github.com/acdlite/react-fiber-architecture). In the future, we intend to replace stack reconciler with it, but at the moment it is far from feature parity.

### Next Steps

Read the [next section](/docs/design-principles.html) to learn about the guiding principles we use for React development.
