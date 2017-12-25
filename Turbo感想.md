# 观Turbo后感

> * 原文地址：[Introducing Turbo: 5x faster than Yarn & NPM, and runs natively in-browser 🔥](https://medium.com/@ericsimons/introducing-turbo-5x-faster-than-yarn-npm-and-runs-natively-in-browser-cc2c39715403)
> * 原文作者：[Eric Simons](https://medium.com/@ericsimons?source=post_header_lockup)
> * 译文地址：[介绍 Turbo：比 Yarn 和 NPM 快 5 倍，可以在本地浏览器中运行](https://juejin.im/post/5a35d58ef265da431a434441?utm_source=gold_browser_extension)

**Turbo** 是我第一次听到的一个名字，原意为 **涡轮增压** 。熟悉赛车的童鞋都知道，运用了涡轮增压技术的车都可以获得更大的动力，跑的更 **快**。

这估计也就是本文即将介绍的Turbo的开发者想要表达的意思吧。

只要进行前端开发，就一定不会对 `node_modules` 这个名字陌生。不知道各位读者有没有经历过这样一件事：网上down了一个项目想要跑起来，首先肯定要运行 `npm i`，就这样一个命令可能需要等很久很久，如果你使用了淘宝源可能还好一点，不过大型项目可能还是要等待一段时间。等待下载完成之后，你就会发现， `node_modules` 已经变得很大很大了。

**Turbo** 做的就是使包的下载变得更快，压缩 `node_modules` 的体积。

- **安装包的速度最少是 Yarn 和 NPM 的五倍 🔥**
- **将  `node_modules`  的大小减少到两个数量级**
- **用于生产级可靠性的多层冗余** 💪
- **完全在 Web 浏览器中工作，能够拥有闪电般的开发环境 ⚡️**

这是原文对 **Turbo** 做的总结。五倍的速度提升，看着真的非常诱人啊。他是怎么做到的呢？

其实每次我们 `npm install ***` 某一个包的时候，是将这个包的所有文件都下载下来，而实际情况我们往往只需要其中一个文件，也就是 `package.json` 中 `main` 字段所对应的文件。我们引用其他包无非就是使用 `require` 或者 `import` ，如果没有指明包中的某一个文件，默认会去查看 `package.json` 中 `main` 字段对应的文件，若没有，默认则为 `index.js` 。而 **Turbo** 则抓住了这一特性，只下载主文件以及被主文件引用的文件，故而包的大小降低了，下载速度也就提高了。

同时 **Turbo** 还提供了缓存机制。如果你需要的包已经存在于他的缓存中，则他会从jsDelivr提供的CDN中下载需要的文件。这种机制无疑再一次的提高了下载速度。

非常可惜的是，由于 **Turbo** 仅仅是针对 *stackblitz*（一个在线IDE）开发的，故无法在本地使用。其实想想也可以理解，如果不是在线IDE，也不会对压缩 `node_modules` 有这么强的需求，对于本地开发来说， `node_modules` 过大，好像也不会影响到什么😜。

不过值得一提的是，文中提到了一种很实用的想法：
> 使用 Turbo 使脚本类型与模块相等

很期待啊，不知道会不会实现，也不知道可不可以移植到本地。