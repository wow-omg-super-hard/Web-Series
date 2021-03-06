[![返回目录](https://parg.co/US3)](https://parg.co/UGZ)

# MobX 底层原理与透明响应式实现

# 单次执行

# 同步推导

# 计算值与响应

# MobX 基础设计理念阐述

数周之前[Bertalan Miklos](https://twitter.com/solkimicreb1)撰写了一篇非常精彩的对比 MobX 与 基于 Proxy 的 NX 系列框架的[博文](http://www.nx-framework.com/blog/public/mobx-vs-nx/)。该文不仅详细解释了基于 Proxy 的响应式框架的原理，还介绍了 MobX 以及背后的 Transparent Reactivity (Automatic Reactivity) 的理念，这其中有很多我本人之前都尚未详细阐述的。本文即是我分享关于 MobX 的独特特性的文章。

# MobX 为何进行同步推导（Derivations Synchronously）

MobX 的重要特性之一即是所有的 Derivations 都是同步计算的，这一点很不同于现有的主流框架。对于 RxJS 这样的 Event Stream 流派的响应式框架，虽然它们也能保证同步推导，但是缺乏了所谓的透明追踪（Transparent Tracking）的特性。而对于 Meteor、Knockout、Angular、Ember 以及 Vue 这些主流的双向数据绑定的 MVVM 框架的响应式表现很类似于 MobX，但是它们在可预测性（Predictability）上的表现差强人意。形象的解释就是如果某个框架重复运行你的代码（譬如重复渲染）或者延迟执行时，开发者很难进行调试，即使是像`Promise`这样的简单的抽象都因为其异步性而难以调试。Flux 或者 Redux 这样的 Action-Dispatcher 框架的核心优势即是在项目动态扩展的同时保证了其可预测性。MobX 则另辟蹊径，没有像 Redux 这样基于纯函数的运行与数据流追踪，而是尝试从根本上去解决这个问题。Transparent Reactivity 本身具有声明式、高阶以及简洁明了的特性，而 MobX 在此之上添加了两个约束：

* 对于任何状态的更改，MobX 会保证所有相关的 Derivation 只会被执行一次。
* Derivations 没有任何的延迟，对于 Observer 而言完全是同步进行的。

约束一着眼于解决多次运行的问题（Double Runs）, 即如果某个推导值依赖于其他推导值，整个 Derivations 会以正确的顺序执行。因此任何的推导值都不会有所谓的过时的、不一致状态出现，关于其实现原理参考之前的[博文](https://medium.com/@mweststrate/becoming-fully-reactive-an-in-depth-explanation-of-mobservable-55995262a254)。约束二：所有的 Derivations 都不会是旧值 则更有意思了。约束二主要是为了解决所谓的临时不一致（Temporary Inconsistencies）的情形，这也是 MobX 采用了与其他 UI 库不一样的更新调度（Scheduling Derivation）策略。目前的 UI 库采取的策略往往是脏检测机制，即首先将发生变化的 Derivations 标记为脏数据，然后在下一轮更新的时候重新运行相关渲染。这种方式简单粗暴，适用于专注 DOM 更新的场景。DOM 本身就具有一定的延迟，我们往往也不会选择在程序里从 DOM 中读取数据，因此临时的数据不一致性（Temporary Staleness）是可以被容忍的。而这种临时的数据不一致性却是任何响应式库的致命缺陷，譬如下面这个例子：

```js
const user = observable({
  firstName: “Michel”,
  lastName: “Weststrate”,
  // MobX computed attribute
  fullName: computed(function() {
    return this.firstName + " " + this.lastName
  })
})
user.lastName = “Vaillant”
sendLetterToUser(user)
```

问题来了，在上述代码中当我们调用`sendLetterToUser(user)`这一句时，我们能够确定读取到的`user`对象中的`fullName`值是更新之后的值还是旧值？在 MobX 中我们能够确定整个推导过程是同步进行的，即永远得到的都是最新的正确的值。这一特性能够保证程序的可预测性，并且 MobX 在调用栈中记录了完整的更新过程，使得调试也变得简单很多。当你想要了解某个值是如何计算而来的，只需要查看整个更新调用栈即可。在 MobX 项目最初启动的时候，很多人都会质疑我们是否能够保证高效地同时达到整个 Derivation Tree 的按序计算与每次更新都会触发属性值的重推导。

# Transactions & Actions

了解 React setState 原理的同学肯定都对其中的事务（Transaction）的概念不陌生，我们应该将多个更新包裹在某个事务内然后一次性计算以提高性能。MobX 中我们同样存在事务的概念，实际上所有的 Derivations 都是在某个事务的结束期进行计算的，不过如果用户在事务结束前就去读取某个属性，那么 MobX 也会保证得到正确的、最新的推导值。MobX 3 中将事务相关的接口修正为了内部接口，而 [actions](https://medium.com/@mweststrate/mobx-2-2-explicit-actions-controlled-mutations-and-improved-dx-45cdc73c7c8d?source=user_profile---------14----------) 本身是会自动包裹在事务内的。Actions 即用于声明某个函数会更新状态，其与 reactions 相辅相成，后者代表某个函数会响应状态的变化。

# Computed Values & Reactions
