---
title: js笔记：Promise, async, await
tags: 前端 js
---

### Promise
Promise本质是一个对象。Promise优化了JS中回调的写法。<br>
1. 一般回调的传统写法是在调用API的时候向其传入一个回调函数指针，然后等待API内部实现在某个时候（一般是执行内部操作成功或失败后）执行这个函数。
2. Promise实现回调的写法是：把回调函数用then()语法绑定到Promise对象上。

举例：<br>
```js
// 成功的回调函数
function successCallback(result) {
  console.log("音频文件创建成功: " + result);
}

// 失败的回调函数
function failureCallback(error) {
  console.log("音频文件创建失败: " + error);
}

// 传统写法
createAudioFileAsync(audioSettings, successCallback, failureCallback)

// promise写法
const promise = createAudioFileAsync(audioSettings);
promise.then(successCallback, failureCallback);
// or 简写成
createAudioFileAsync(audioSettings).then(successCallback, failureCallback);

```

### Promise约定
1. 在本轮runloop运行完成之前，回调函数是不会被调用的。
2. 即使异步操作已经完成（成功或失败），在这之后通过 then() 添加的回调函数也会被调用。
3. 通过多次调用 then() 可以添加多个回调函数，它们会按照插入顺序进行执行。

### Promise链式调用
Promise 很棒的一点就是链式调用（chaining）。<br>
```js
// successCallback, failureCallback函数的返回结果也是一个Promise对象
const promise = doSomething();
const promise2 = promise.then(successCallback, failureCallback);
// or
const promise2 = doSomething().then(successCallback, failureCallback);
```
经典的地狱回调
```js
doSomething(function(result) {
  doSomethingElse(result, function(newResult) {
    doThirdThing(newResult, function(finalResult) {
      console.log('Got the final result: ' + finalResult);
    }, failureCallback);
  }, failureCallback);
}, failureCallback);
```
用Promise重写地狱回调
```js
doSomething().then(function(result) {
  return doSomethingElse(result);
})
.then(function(newResult) {
  return doThirdThing(newResult);
})
.then(function(finalResult) {
  console.log('Got the final result: ' + finalResult);
})
.catch(failureCallback);

// 用箭头函数
doSomething()
.then(result => doSomethingElse(result))
.then(newResult => doThirdThing(newResult))
.then(finalResult => {
  console.log(`Got the final result: ${finalResult}`);
})
.catch(failureCallback);

```

### Promise举例
```js
function myAsyncFunction(url) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("GET", url);
    xhr.onload = () => resolve(xhr.responseText);
    xhr.onerror = () => reject(xhr.statusText);
    xhr.send();
  });
};
```

### async
简单来说，async&wait是基于promises的语法糖，使异步代码更易于编写和阅读。通过使用它们，异步代码看起来更像是老式同步代码。<br>
我们使用 async 关键字，把它放在函数声明之前，使其成为 async function，一个async function的返回值一定是一个Promise。
```js
// 普通函数
function hello() { return "Hello" };
hello(); // 输出hello

// async函数
async function hello() { return "Hello" };
hello(); // 输出Promise {<fulfilled>: "Hello"}，因为返回值是一个promise
hello().then((value) => console.log(value)) // 输出hello

```

### await
await只在异步函数里面才起作用。它可以放在任何异步的，基于 promise 的函数之前。它会暂停代码在该行上，直到 promise 完成，然后返回结果值。
```js
async function myFetch() {
  let response = await fetch('coffee.jpg');
  let myBlob = await response.blob();

  let objectURL = URL.createObjectURL(myBlob);
  let image = document.createElement('img');
  image.src = objectURL;
  document.body.appendChild(image);
}

myFetch()
.catch(e => {
  console.log('There has been a problem with your fetch operation: ' + e.message);
});
```

Promise.all:
```js
async function fetchAndDecode(url, type) {
  let response = await fetch(url);

  let content;

  if(type === 'blob') {
    content = await response.blob();
  } else if(type === 'text') {
    content = await response.text();
  }

  return content;
}

async function displayContent() {
  let coffee = fetchAndDecode('coffee.jpg', 'blob');
  let tea = fetchAndDecode('tea.jpg', 'blob');
  let description = fetchAndDecode('description.txt', 'text');

  let values = await Promise.all([coffee, tea, description]);

  let objectURL1 = URL.createObjectURL(values[0]);
  let objectURL2 = URL.createObjectURL(values[1]);
  let descText = values[2];

  let image1 = document.createElement('img');
  let image2 = document.createElement('img');
  image1.src = objectURL1;
  image2.src = objectURL2;
  document.body.appendChild(image1);
  document.body.appendChild(image2);

  let para = document.createElement('p');
  para.textContent = descText;
  document.body.appendChild(para);
}

displayContent()
.catch((e) =>
  console.log(e)
);
```

