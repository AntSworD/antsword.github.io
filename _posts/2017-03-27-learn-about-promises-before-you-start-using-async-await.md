---
layout: post
title: 想用`async/await`？先去学`Promise`!
---

随着 ES2016（或者说 ES7）的到来，Babel 现在已经支持 async/await 了，越来越多的人开始意识到这样通过同步代码的结构来编写异步代码的模式简直是棒极了。

这是一个对代码质量有着巨大提升的好事。

然而，却有很多人对整个 async/await 原理的基础知识 — **`Promise`** 不甚了解。实际上**你写的每个`async`函数都是返回一个`Promise`实例，而每个`await`的对象都应该是个`Promise`实例**。

为什么我一直强调这一点呢？因为现在很多的`javascript`书面资料还都是在使用`callback`的形式来写异步代码；很多人并没有正式学习过`Promise`的知识，所以他们错过了`async/await`知识中很重要的一个点。

## 什么是`Promise`?
我会尽量简洁地进行解释，因为关于这个的解释已经被在很多文章中都有说到了。

一个`promise`对象是一种*包含*另一个对象的特殊的`javascript`对象。我有个`promise`对象，它可以包含数字 17、字符串”hello world”又或者是一些任意的对象，或者是*其他任何*你可以用 `javascript` 变量正常存储的东西。

那我又应该如何访问`promise`对象所包含的数据呢？使用`.then()`：

	function getFirstUser() {
	  return getUsers().then(function(users) {
	    return users[0].name;
	  });
	}

然后我又应该怎么捕获`promise`链上面的错误呢？可以使用`.catch()`

	function getFirstUser() {
	  return getUsers().then(function (users) {
	    return users[0].name;
	  }).catch(function (err) {
	    return {
	      name: 'default user'
	    };
	  });
	}

虽然`promise`通常代表的是『未来』的数据，但是只要我一旦有了个可以返回某些数据的`promise`对象，我就已经不关心数据到底是未来的数据还是已经得到的数据，在任何情况下我都只需要调用`.then()`就可以得到结果就可以了。因此，可以说`promise`是强一致异步的，换句话说就是：『无论返回值是当前可用还是未来可用，这(`promise`)都是个异步函数』。

## Cool…`async/await`又是如何配合使用呢？

好的，让我们继续研究上面的代码。我们现在已经知道，`getUsers()`返回了一个`promise`对象。而只要我们使用 ES2016 语法，就可以对得到的任何`promise`进行`await`。所有`await`在字面上的意思都是：在`promise`上使用时，功能完全等同于调用了`.then()`（但是不需要任何回调函数）。所以，上面的代码就变成了：

	async function getFirstUser() {
	  let users = await getUsers();
	  return users[0].name;
	}

我可以`await`任何我想要`await`的`promise`，无论这个`promise`是否已经完成，无论(用户)是否已经创建。`await`会简单地暂停我的函数的执行，直到`promise`已经完成操作，成功返回了结果数据。

然后，我们又要怎么去捕获错误呢？

这个非常简单，可以看到，我们现在写的代码已经是同步风格的了，我们已经可以重新开始使用`try/catch`了：

	async function getFirstUser() {
	  try {
	    let users = await getUsers();
	    return users[0].name;
	  } catch (err) {
	    return {
	      name: 'default user'
	    };
	  }
	}

这就完成了，是不是非常帅！到这里，我们已经可以通过`Promsise`编写异步操作了，而且可以用`async/await`来很好地编写我们的代码了。可是，我为什么又要说要重新关注`Promise`呢？

## 陷阱 1：没有 使用`await`
 如果我一不小心这样调用：

	let user = getFirstUser();

即使我的代码中没有`await`，程序也不会自动报错！

事实上，我没有严格按照要求去`await`任何东西。但是，如果我没有使用`await`，`user`将会是一个`promise`对象的引用（而不是一个已完成操作的结果值），而且我对这个失误却是无能为力。如果我一开始没有认真地编写我的`javascript`代码，在我尝试使用这个`user`变量（是一个`promise`对象而不是`user`数据）做其他操作前，我并不能明显地发现这一点，更别说这个代码报错的指向可能在我非常意外的某些无关代码上面。

所以，我们要记住这个很重要的点：`async`函数是不会神奇的去`await`他们自己的。*除非你期望得到的就是一个`promise`对象，否则你必须使用`await`*。

即使如此，只要你确实希望的是一个`promise`对象，这便是一个好特性。这会给我们更多的控制力去做些更酷炫的操作，就像[  memoizing promises ](https://medium.com/@bluepnume/async-javascript-is-much-more-fun-when-you-spend-less-time-thinking-about-control-flow-8580ce9f73fc#.v0igtb4b7 " memoizing promises")里面做的。

## 陷阱 2：`await`多个异步操作的返回
这里要讲的是`await`的一个小问题：在正常情况下使用时，我可以一次进行一个`await`：

	let foo = await getFoo();
	let bar = await getBar();

即使我本来是想同时得到两个结果，程序会按顺序得到`foo`和`bar`。

对于这个问题，有个已经被否决的解决方法[ rejected from the ES2016 spec](https://github.com/tc39/ecmascript-asyncawait/issues/61 "rejected from the ES2016 spec")：

	let [foo, bar] = await* [getFoo(), getBar()];

这实在有点尴尬，因为这确实是个解决问题的好办法。现在取而代之的是下面这种方法：

	let [foo, bar] = await Promise.all([getFoo(), getBar()]);

这是个让人有点混乱的做法：我们不是在用`async/await`么？不是说不用`promise`么？！这个做法会奏效的原因是因为，*`async/await`和`promise`在底层是相同的东西。*

如果你理解了`Promise.all`的意义，对于掌握这一点要容易得多，所以，让我们回到`promise`的基础知识上。从根本上来说，`Promise.all`会接收一个子元素都是`promise`的*数组*，然后把它们合并成**只有在数组里每个子`promise`都已经完成操作后才会返回**的*一个* `promise`。

然后，在上面的例子中，我们`await`的是我们`Promise.all`创造出的「超级`promise`」。所以，在你在代码里使用`async/await`之前，理解了`Promise.all`的含义是非常有帮助的。

在我们讨论这个话题的时候，我想要搞清楚我注意到的一个常见的误解。`Promise.all`*并不是在「调度（dispatch）」*或者「创建」你传进去的那些`promise`。在我创建数组的时候 —

	[getFoo(), getBar()]

— 这些异步操作*已经在进行中了*。所有`Promise.all`做的是把它们聚合到一个单独的新`promise`，然后等待它们全部完成操作。`Promise.all`并*没有*「做这些操作」，而是在「等待这些操作」。这个和实实在在地调度你传进去的那些方法的[`async.parallel`](http://caolan.github.io/async/docs.html#parallel)是完全不同的。

—

出于兴趣的缘故，下面是另一个使用`async/await`来处理并行任务的有趣方法（虽然我并不推荐）：

	let fooPromise = getFoo();
	let barPromise = getBar();
	let foo = await fooPromise;
	let bar = await barPromise;

这个代码再次挑战了你对`promise`的理解，是否真正的了解其中发生了什么。

1. 首先，我们调用了`getFoo`和`getBar`，并把它们返回的`promise`保存在`fooPromise `和`barPromise`.
2. 这些异步操作现在已经在执行了，它们正在发生，并没有暂停或者延迟这些操作。
3. 我们依次`await`这些异步操作的完成。

难道这个`await`并不是说这两个异步操作是串行执行的？当然不是！在这个例子中，我们在`await`它们前就已经对这*两个*异步操作进行了调用执行，所以这两个异步操作是同时开始执行的。当我们开始`await`，想延时执行这两行异步操作的时候，已经太晚了。

显而易见，千万别这样做！这可读性实在太差了。但这是一个演示了为什么在我们使用`async/await`的代码里还要用到`promise`的例子。

## 陷阱3：你的整个执行栈都改成`async`
如果我在某些地方使用`await`，有个问题就是我的整个执行栈都会受到影响。为了调用我的一个`async`函数，理想状态下调用者本身就要是一个`async`函数。在我的执行栈这是个连锁反应，这让我很难去把我的`callback`代码逐步转换成`async/await`。

注意：如果你已经在代码中使用了`promise`，情况是不一样的，因为，记住—`async`函数返回的就是这些可以被`await`的`promise`，所以使用了`promise`后，你已经在这方面有了90％的兼容性

这里要说的是：如果你了解`promise`是怎么工作的，你可以通过把`promise`改造成一个接受回调函数的异步函数来使用来解决这个问题。思考下下面的代码：

	function getFirstUser(callback) {
	  return getUsers().then(function(user) {
	    return callback(null, users[0].name);
	  }).catch(function(err) {
	    return callback(err);
	  });
	}

就是这样，我刚刚把一个`async`函数（`getUsers`）转换成一个通过`callback`返回没有太多冗余的结果的函数。事实上，很多`promise`库甚至通过一个`nodeify()`方法帮你完成这个工作：

	function getFirstUser(callback) {
	  return getUsers().then(function(user) {
	    return users[0].name;
	  }).nodeify(callback);
	}

无论如何，还有其他更好的方式么？如果我需要在`async`函数里调用一些通过`callback`返回的函数呢？

同样的，这个有助于我们对`promise`的理解，因为真正唯一的解决办法是把我们的回调函数转换成返回`promise`的函数 — 使用 ES6 语法，把一个`callback`转换成`promise`是非常简单的：

	function callbackToPromise(method, ...args) {
	  return new Promise(function(resolve, reject) {
	    return method(...args, function(err, result) {
	      return err ? reject(err) : resolve(result);
	    });
	  });
	}

然后：

	async function getFirstUser() {
	  let users = await callbackToPromise(getUsers);
	  return users[0].name;
	}

## 陷阱4：要记得处理错误
这个问题不仅是`promise`的老问题，同样也困扰着`async/await`：你一定要记得去捕获错误，否则这些错误可能会遗失在未知的远方。

思考下面的代码：

	myApp.endpoint('GET', '/api/firstUser', async function(req, res) {
	  let firstUser = await getFirstUser();
	  res.json(firstUser)
	});

如果`myApp.endpoint`（这个方法可能不是你编写的）不是个`promise`或者`async`感知的，没有在我传进去的处理函数上调用了`await`或者`.catch()`，所有错误都会丢失。

只要你想象下下面这个用`promise`写的相同作用的代码，你就会马上发现原因：

	myApp.endpoint('GET', '/api/firstUser', function(req, res) {
	  return getFirstUser().then(function(firstUser) {
	    res.json(firstUser)
	  });
	});

在这里我们在回调函数里传了一个`getFirstUser`*成功*返回的处理逻辑，但是我们并没有对*错误*进行任何处理。因为`getFirstUser `是个异步函数，即使它抛出了一个错误，也不会被`myApp.endpoint`给自动捕获到。

因此，你要把你的代码逻辑包含在一个`try/catch`（必须捕获所有错误）里，确保你可以处理任何抛出的错误：

	myApp.registerEndpoint('GET', '/api/firstUser', async function(req, res) {
	  try {
	    let firstUser = await getFirstUser();
	    res.json(firstUser)
	  } catch (err) {
	    console.error(err);
	    res.status(500);
	  }
	});

希望不久以后，越来越多的框架可以是`async/await`（或者说`promise`）感知的，这可以让我们更少地关注这个问题。

— — —

那么，如果从这一切来怎么看呢？

**只有你理解了`promise`，你才能更好地解决这些使用`async/await`过程中遇到的实在难以理解的案例和`bug`。**

另外，这里还有个学习`Promise`的好处，即使你不喜欢用`babel`，现在也可以通过把你的代码改造成`promise`，使其以后可以完全从 ES2016 的`async/await`语法中受益（译者注：`Node.JS`在`v7.6.0`已经支持`async/await`）。喔！

[从这里查看这篇文章的续篇](https://medium.com/@bluepnume/even-with-async-await-you-probably-still-need-promises-9b259854c161#.gulpu2ykc)

Source Link: [Understand promises before you start using async/await](https://medium.com/@bluepnume/learn-about-promises-before-you-start-using-async-await-eb148164a9c8#.2lyynbfhm)
