---
title: Making setInterval Declarative with React Hooks
date: '2019-02-04'
spoiler: How I learned to stop worrying and love refs.
---

接触 [React Hooks](https://reactjs.org/docs/hooks-intro.html) 一定时间的你，也许会碰到一个神奇的问题: `setInterval` [用起来没你想的简单](https://stackoverflow.com/questions/53024496/state-not-updating-when-using-react-state-hook-within-setinterval)。

Ryan Florence 在[他的推文](https://mobile.twitter.com/ryanflorence/status/1088606583637061634)里面说到：

> 不少朋友跟我提起，setInterval 和 hooks 一起用的时候，有种蛋蛋的忧伤。

老实说，这些朋友也不是胡扯。刚开始接触 Hooks 的时候，*确实*还挺让人疑惑的。

但我认为谈不上 Hooks 的毛病，而是 [React 编程模型](https://overreacted.io/react-as-a-ui-runtime/)和 `setInterval` 之间的一种模式差异。相比类（Class），Hooks 更贴近 React 编程模型，使得这种差异更加突出。

**虽然有点绕，但是让两者和谐相处的方法，还是有的。**

本文就来探索一下，*如何*让 setInterval 和 Hooks 和谐地玩耍，*为什么*是这种方式，以及这种方式给你带来了什么*新能力*。

-----

**声明：本文采用循序渐进的示例来解释问题。所以有一些示例虽然看起来可以有捷径可走，但是我们还是一步步来。**

如果你是 Hooks 新手，不太明白我在纠结啥，不妨读一下 React Hooks 的[介绍](https://medium.com/@dan_abramov/making-sense-of-react-hooks-fdbde8803889)和[官方文档](https://reactjs.org/docs/hooks-intro.html)。本文假设读者已经使用 Hooks 超过一个小时。

---

## 代码呢？

通过下面的方式，我们可以轻松地实现一个每秒自增的计数器：

```jsx{6-9}
import React, { useState, useEffect, useRef } from 'react';

function Counter() {
  let [count, setCount] = useState(0);

  useInterval(() => {
    // Your custom logic here
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}
```

*（[CodeSandbox 线上示例](https://codesandbox.io/s/105x531vkq)）*

上述 `useInterval` 并不是内置的 React Hook，而是我实现的一个[自定义 Hook](https://reactjs.org/docs/hooks-custom.html)：

```jsx
import React, { useState, useEffect, useRef } from 'react';

function useInterval(callback, delay) {
  const savedCallback = useRef();

  // Remember the latest callback.
  useEffect(() => {
    savedCallback.current = callback;
  });

  // Set up the interval.
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }
    if (delay !== null) {
      let id = setInterval(tick, delay);
      return () => clearInterval(id);
    }
  }, [delay]);
}
```

*（如果你在错过了，这里也有一个一样的 [CodeSandbox 线上示例](https://codesandbox.io/s/105x531vkq)）*

**我实现的 `useInterval` Hook 设置了一个计时器，并且在组件 unmount 的时候清理掉了。** 这是通过组件生命周期上绑定 `setInterval` 与 `clearInterval` 的组合完成的。

这是一份可以在项目中随意复制粘贴的实现，你甚至可以发布到 NPM 上。

**不关心为什么这样实现的读者，就不用继续阅读了。下面的内容是为希望深入理解 React Hooks 的读者而准备的。**

---

## 哈？！ 🤔

我知道你想什么：

> Dan，这代码不对劲。说好的“纯粹 JavaScript”呢？React Hooks 打了 React 哲学的脸？

**哈，我一开始也是这么想的，但是后来我改观了，现在，我准备也改变你的想法**。开始之前，我先介绍下这份实现的能力。

---

## 为什么 `useInterval()` 是一个更合理的 API？

注意下，`useInterval` Hook 接收一个函数和一个延时作为参数：

```jsx
  useInterval(() => {
    // ...
  }, 1000);
```

这个跟原生的 `setInterval` 非常的相似：

```jsx
  setInterval(() => {
    // ...
  }, 1000);
```

**那为啥不干脆使用 `setInterval` 呢？**

`setInterval` 和 `useInterval` Hook 最大的区别在于，`useInterval` Hook 的**参数是“动态的”**。乍眼一看，可能不是那么明显。

我将通过一个实际的例子来说明这个问题：

---

如果我们希望 interval 的间隔是可调的：

![一个延时可输入的计时器](https://overreacted.io/counter_delay-35e4f35a8585255b11c090aed9f72433.gif)

此时无需手动控制延时，直接动态调整 Hooks 参数就行了。比方说，我们可以在用户切换到另一个选项卡时，降低 AJAX 更新数据的频率。

如果按照类（Class）的方式，怎么通过 `setInterval` 实现上述需求呢？我折腾出这个：

```jsx{7-26}
class Counter extends React.Component {
  state = {
    count: 0,
    delay: 1000,
  };

  componentDidMount() {
    this.interval = setInterval(this.tick, this.state.delay);
  }

  componentDidUpdate(prevProps, prevState) {
    if (prevState.delay !== this.state.delay) {
      clearInterval(this.interval);
      this.interval = setInterval(this.tick, this.state.delay);
    }
  }

  componentWillUnmount() {
    clearInterval(this.interval);
  }

  tick = () => {
    this.setState({
      count: this.state.count + 1
    });
  }

  handleDelayChange = (e) => {
    this.setState({ delay: Number(e.target.value) });
  }

  render() {
    return (
      <>
        <h1>{this.state.count}</h1>
        <input value={this.state.delay} onChange={this.handleDelayChange} />
      </>
    );
  }
}
```

*([CodeSandbox 在线示例](https://codesandbox.io/s/mz20m600mp))*

太熟悉了！

那改成使用 Hooks 怎么实现呢？

<font size="50">🥁🥁🥁开始表演了！</font>

```jsx{5-8}
function Counter() {
  let [count, setCount] = useState(0);
  let [delay, setDelay] = useState(1000);

  useInterval(() => {
    // Your custom logic here
    setCount(count + 1);
  }, delay);

  function handleDelayChange(e) {
    setDelay(Number(e.target.value));
  }

  return (
    <>
      <h1>{count}</h1>
      <input value={delay} onChange={handleDelayChange} />
    </>
  );
}
```

*([CodeSandbox 线上示例](https://codesandbox.io/s/329jy81rlm))*

没了，就这么多！

不用于 class 实现的版本，`useInterval` Hook “升级到”支持到支持动态调整延时的版本，没有增加任何复杂度。

使用 `useInterval` 新增动态延时能力，几乎没有增加任何复杂度。这个优势是使用 class 无法比拟的。

```jsx{4,9}
// 固定延时
useInterval(() => {
  setCount(count + 1);
}, 1000);

// 动态延时
useInterval(() => {
  setCount(count + 1);
}, delay);
```

当 `useInterval` 接收到另一个 delay 的时候，它就会重新设置计时器。

**我们并没有通过执行代码来*设置*或者*清理*计时器，而是*声明*了具有特定延时的计时器 - 这是我们实现的 `useInterval` 的根本原因。**

如果想临时*暂停*计时器呢？我可以这样来：

```jsx{6}
const [delay, setDelay] = useState(1000);
const [isRunning, setIsRunning] = useState(true);

useInterval(() => {
  setCount(count + 1);
}, isRunning ? delay : null);
```

*([线上示例](https://codesandbox.io/s/l240mp2pm7))*

这就是 Hooks 和 React 再一次让我兴奋的原因。我们可以把原有的调用式 API，包装成声明式 API，从而更加贴切地表达我们的意图。就跟渲染一样，我们可以**描述当前时间每个点的状态**，而无需小心翼翼地通过具体的命令来操作它们。

---

到这里，我希望你已经确信 `useInterval` Hook 是一个更好的 API - 至少在组件层面使用的时候是这样。

**可是为什么在 Hooks 里使用 `setInterval` 和 `clearInterval` 这么让人恼火？** 回到刚开始的计时器例子，我们尝试手动去实现它。

---

## 第一次

最简单的，渲染初始状态:

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <h1>{count}</h1>;
}
```

现在我希望它每秒定时更新。我准备使用 `useEffect()` 并且返回一个清理方法，因为它是一个[需要清理的 Side Effect](https://reactjs.org/docs/hooks-effect.html#effects-with-cleanup)：

```jsx{4-9}
function Counter() {
  let [count, setCount] = useState(0);

  useEffect(() => {
    let id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  });

  return <h1>{count}</h1>;
}
```

*(查看 [CodeSandbox 线上示例](https://codesandbox.io/s/7wlxk1k87j))*

看起来很简单？

**然而，这段代码有个诡异的行为。**

React 默认会在每次渲染时，都重新执行 effects。这是符合预期的，这机制规避了早期在 React Class 组件中存在的[一系列问题](https://reactjs.org/docs/hooks-effect.html#explanation-why-effects-run-on-each-update)。

通常来说，这是一个好特性，因为大部分的订阅 API 都允许移除旧的订阅并添加一个新的订阅来替换。但是，这不包括 `setInterval`。调用了 `clearInterval` 后重新 `setInterval` 的时候，计时会被重置。如果我们频繁重新渲染，导致 effects 频繁执行，计时器可能根本没有机会被触发！

通过使用在一个**更小的时间间隔**重新渲染我们的组件，可以重现这个 BUG：

```jsx
setInterval(() => {
  // 重新渲染导致的 effect 重新执行会让计时器在调用之前，
  // 就被 clearInterval() 清理掉，之后 setInterval()
  // 重新设置的计时器，会重新开始计时
  ReactDOM.render(<Counter />, rootElement);
}, 100);
```

*(查看这个 BUG 的[线上示例](https://codesandbox.io/s/9j86r218y4))*

---

## 第二次

部分读者可能知道，`useEffect` 允许我们[控制重新执行的实际](https://reactjs.org/docs/hooks-effect.html#tip-optimizing-performance-by-skipping-effects)。通过在第二个参数指定依赖数组，React 就会只在这个依赖数组变更的时候重新执行 effect。

```jsx{3}
useEffect(() => {
  document.title = `You clicked ${count} times`;
}, [count]);
```

如果我们希望 effect **只在**组件 mount 的时候执行，并且在 unmount 的时候清理，我们可以传递空数组 `[]` 作为依赖。

但是！不是特别熟悉 JavaScript 闭包的读者，很可能会犯一个共性错误。我来示范一下！（我们在设计 [lint 规则](https://github.com/facebook/react/pull/1463)来帮助定位此类错误，不过现在还没有准备好。）

第一次的问题在于，effect 的重新执行导致计时器太早被清理掉了。如果不重新执行它们，也许可以解决这个问题：

```jsx{9}
function Counter() {
  let [count, setCount] = useState(0);

  useEffect(() => {
    let id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

如果这样实现，计时器更新到 1 之后，就停止不动了。([查看这个 BUG 的线上示例](https://codesandbox.io/s/jj0mk6y683))

发生了啥？

**问题在于，`useEffect` 使用的 `count` 是在第一次渲染的时候获取的。** 获取的时候，它就是 `0`。由于一直没有重新执行 effect，所以 `setInterval` 在闭包中使用的 `count` 始终是从第一次渲染时来的，所以就有了 `count + 1` 始终是 `1` 的现象。呵呵哒！ 

**我感觉你已经开始怼天怼地了。Hooks 是什么鬼嘛！**

解决这个问题的[一个方案](https://codesandbox.io/s/j379jxrzjy)，是把 `setCount(count + 1)` 替换成“更新回调”的方式 `setCount(c => c + 1)`。从回调参数中，可以获取到最新的状态。此非万全之策，新的 props 就无法读取到。

[另一个解决方案](https://codesandbox.io/s/00o9o95jyv)是使用 [`useReducer()`](https://reactjs.org/docs/hooks-reference.html#usereducer)。此方案更为灵活。在 reducer 内部，可以访问当前的状态，以及最新的 props。`dispatch` 方法本身不会改变，所以你可以在闭包里往里面灌任何数据。使用 `useReducer()` 的一个限制是，你不能在内部触发 effects。（不过，你是可以通过返回一个新 state 来触发一些 effect）。

**为何如此艰难？**

---

## 阻抗不匹配

这个术语（*译者注：术语原文为 "Impedance Mismatch"*）在很多地方被大家使用，[Phil Haack](https://haacked.com/archive/2004/06/15/impedance-mismatch.aspx/) 是这样解释的：

> 有人说数据库来自火星，对象来自金星。数据库不能天然的和对象模型建立映射关系。这就像尝试将两块磁铁的 N 极挤在一起一样。

我们此处的“阻抗不匹配”，说的不是数据库和对象。而是 React 编程模型，与命令式的 `setInterval` API 之间的不匹配。

**一个 React 组件可能会被 mount 一段时间，并且经历多个不同的状态，不过它的 render 结果*一次性*地描述了*所有*这些状态**

```jsx
// 描述了每一次渲染的状态
return <h1>{count}</h1>
```

同理，Hooks 让我们声明式地使用一些 effect：

```jsx{4}
// 描述每一个计数器的状态
useInterval(() => {
  setCount(count + 1);
}, isRunning ? delay : null);
```

我们不需要去*设置*计时器，但是指明了它*是否*应该被设置，以及设置的间隔是多少。我们事先的 Hook 就是这么做的。通过离散的声明，我们描述了一个连续的过程。

**相对应的，`setInterval` 却没有描述到整个过程 - 一旦你设置了计时器，它就无法改变了，只能清除它。**

这就是 React 模型和 `setInterval` API 之间的“阻抗不匹配”。

---

React 组件的 props 和 state 会变化时，都会被重新渲染，并且把之前的渲染结果“忘记”的一干二净。两次渲染之间，是互不相干的。

`useEffect()` Hook 同样会“遗忘”之前的结果。它清理上一个 effect 并且设置新的 effect。新的 effect 获取到了新的 props 和 state。所以我们[第一次](https://codesandbox.io/s/7wlxk1k87j)的事先在某些简单的情况下，是可以执行的。

**但是 `setInterval()` 不会 “忘记”。** 它会一直引用着旧的 props 和 state，除非把它换了。但是只要把它换了，就没法不重新设置时间了。

等会，真的不能吗?

---

## Refs 是救星!

先把问题整理下：

* 第一次渲染的时候，使用 `callback1` 进行 `setInterval(callback1, delay)`
* 下一次渲染的时候，使用 `callback2` 可以访问到新的 props 和 state
* 我们无法用 callback2 替换掉 callback1 但是又不重设计时器

**如果我们压根不替换计时器，而是传入一个 `savedCallback` 变量，始终指向*最新*的计时器回调呢？？**

现在我们的方案看起来是这样的：

* 设置计时器 `setInterval(fn, delay)`，其中 `fn` 调用 `savedCallback`。
* 第一次渲染，设置 `savedCallback` 为 `callback1`
* 第二次渲染，设置 `savedCallback` 为 `callback2`
* ???
* 行了

可变的 `savedCallback` 需要在多次渲染之间“持久化”，所以不能使用常规变量。我们需要像类似实例字段的手段。

[从 Hooks 的 FAQ 中](https://reactjs.org/docs/hooks-faq.html#is-there-something-like-instance-variables)，我们得知 `useRef()` 可以帮我们做到这点：

```jsx
const savedCallback = useRef();
// { current: null }
```

*（你可能已经对 React 的 [DOM refs](https://reactjs.org/docs/refs-and-the-dom.html) 比较熟悉了。Hooks 引用了相同的概念，用于持有任意可变的值。一个 ref 就行一个“盒子”，可以放东西进去。）*

`useRef()` 返回了一个字面量，持有一个可变的 `current` 属性，在每一次渲染之间共享。我们可以把*最新*的计时器回调保存进去。

```jsx{8}
function callback() {
  // 可以读取到最新的 state 和 props
  setCount(count + 1);
}

// 每次渲染，保存最新的回调到 ref 中
useEffect(() => {
  savedCallback.current = callback;
});
```

后续就可以在计时器回调中调用它了：

```jsx{3,8}
useEffect(() => {
  function tick() {
    savedCallback.current();
  }

  let id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);
```

由于传入了 `[]`，我们的 effect 不会重新执行，所以计时器不会被重置。另一方面，由于设置了 `savedCallback` ref，我们可以获取到最后一次渲染时设置的回调，然后在计时器触发时调用。

再看一遍完整的实现：

```js{10,15}
function Counter() {
  const [count, setCount] = useState(0);
  const savedCallback = useRef();

  function callback() {
    setCount(count + 1);
  }

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

*(查看 [CodeSandbox 线上示例](https://codesandbox.io/s/3499qqr565))*

---

## 提取为自定义 Hook

不得不承认，上面的代码有点迷。各种花里胡哨的操作让人费解不说，还有可能让 state 和 refs 与其它逻辑里的搞混。

**我认为，虽然 Hooks 相比 Class 提供了更底层的能力 - 不过 Hooks 的牛逼在于允许我们重组、抽象后创造出声明语意更优的 Hooks**

事实上，我就想这样来写：

```js{4-6}
function Counter() {
  const [count, setCount] = useState(0);

  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}
```

于是我把我的实现核心拷贝到自定义 Hook 中：

```jsx
function useInterval(callback) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);
}
```

延时值 `1000` 是硬编码的，把它参数化：

```jsx
function useInterval(callback, delay) {
```

在设置计时器的时候使用：

```jsx
let id = setInterval(tick, delay);
```

现在 `delay` 可能在多次渲染之间变更，我需要把它声明为计时器 effect 的依赖：

```js{8}
useEffect(() => {
  function tick() {
    savedCallback.current();
  }

  let id = setInterval(tick, delay);
  return () => clearInterval(id);
}, [delay]);
```

慢着，我们之前不是为了避免计时器重设，才传入了一个 `[]` 的吗？不完全是。我们只是希望 Hooks 不要在 *callback* 变更的重新执行。如果 `delay` 变更了，我们是*想要*重新启动计时器的。

现在来看下我们的代码是不是能跑：

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}

function useInterval(callback, delay) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

*(读者可以在 [CodeSandbox](https://codesandbox.io/s/xvyl15375w) 上试一下)*

棒棒的！现在，我们可以无需关注实现细节，在任何组件里面需要的时候，直接使用 `useInterval()` 了。

## Bonus: 暂停计时器

我们希望在给 `delay` 传 `null` 的时候暂停计时器：

```jsx{6}
const [delay, setDelay] = useState(1000);
const [isRunning, setIsRunning] = useState(true);

useInterval(() => {
  setCount(count + 1);
}, isRunning ? delay : null);
```

怎么实现？简单：不设置计时器就可以了。

```js{6}
useEffect(() => {
  function tick() {
    savedCallback.current();
  }

  if (delay !== null) {
    let id = setInterval(tick, delay);
    return () => clearInterval(id);
  }
}, [delay]);
```

*([CodeSandbox 线上示例](https://codesandbox.io/s/l240mp2pm7))*

就这样了。这段代码可以处理各种可能的变更了：延时值改变、暂停和继续。虽然 `useEffect()` API 需要我们前期花更多的精力进行设置和清理工作，添加新能力却是轻松了。

## Bonus: 有趣的 Demo

这个 `useInterval()` Hook 其实很好玩。现在 side effects 是声明式的，所以组合使用变得轻松多了。

**比方说，我们可以使用一个计时器来控制另一个计时器的 `delay`：**

![自动加速的计时器](https://overreacted.io/counter_inception-10cfc4b38497a46980d3a13048a56e36.gif)

```js{10-15}
function Counter() {
  const [delay, setDelay] = useState(1000);
  const [count, setCount] = useState(0);

  // Increment the counter.
  useInterval(() => {
    setCount(count + 1);
  }, delay);

  // Make it faster every second!
  useInterval(() => {
    if (delay > 10) {
      setDelay(delay / 2);
    }
  }, 1000);

  function handleReset() {
    setDelay(1000);
  }

  return (
    <>
      <h1>Counter: {count}</h1>
      <h4>Delay: {delay}</h4>
      <button onClick={handleReset}>
        Reset delay
      </button>
    </>
  );
}
```

*([CodeSandbox 线上示例](https://codesandbox.io/s/znr418qp13))*

## 总结

Hooks 需要我们慢慢适应 - *尤其是*在面对命令式和声明式代码的区别时。你可以创造出像 [React Spring](http://react-spring.surge.sh/docs/hooks) 一样强大的声明式抽象，但是他们复杂的用法偶尔会让你紧张。

Hooks 还很年轻，还有很多我们可以研究和对比的模式。如果你习惯于按照“最佳实践”来的话，大可不必着急使用 Hooks。社区还需时间来尝试和挖掘更多的内容。

使用 Hooks 的时候，涉及到类似 `setInterval()` 的 API，会碰到一些问题。阅读本文后，希望读者能够理解并且解决它们，同时，通过创建更加语义化的声明式 API，享受其带来的好处。
