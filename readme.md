# class组件（class components）里3种绑定this的写法
本总结源于[官方文档](https://reactjs.org/docs/handling-events.html)
先贴出网站的源码
```
class Toggle extends React.Component {
  constructor(props) {
    super(props);
    this.state = {isToggleOn: true};

    // This binding is necessary to make `this` work in the callback
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState(prevState => ({
      isToggleOn: !prevState.isToggleOn
    }));
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        {this.state.isToggleOn ? 'ON' : 'OFF'}
      </button>
    );
  }
}

ReactDOM.render(
  <Toggle />,
  document.getElementById('root')
);
```
这里handleClick使用handleClick(){}定义成了一个类方法，其存储位置在原型上，然后在constructor里用this.handleClick定义了一个实例变量。
首先，这2个变量（函数）是同名的，都叫handleClick，但性质不一样，一个是类方法，一个是实例变量。在调用handleClick的时候，先在实例变量里寻找handleClick，如果没有找到再到原型链上去寻找（类方法），所以并不矛盾。这里，将始终使用实例变量里的handleClick。
其次，实例变量里的handleClick使用了[bind语法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)，这个调用会返回一个新的函数的引用，并且这个新函数绑定了我们所需要的那个this即Toggle对象。如果不这样做，直接使用类方法handleClick，因为类方法的this是在调用的时候才确定的，那么就会出现this不对的问题。
以上是使用bind语法的写法，官网还介绍了另2个。
```
class LoggingButton extends React.Component {
  // This syntax ensures `this` is bound within handleClick.
  // Warning: this is *experimental* syntax.
  handleClick = () => {
    console.log('this is:', this);
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        Click me
      </button>
    );
  }
}
```
这是使用字段属性的语法，因为字段是属于实例的即便它是一个函数，所以这里的this必然绑定到类本身。注意这里的写法，是一个箭头函数。官网也说了，这个字段属性的语法并不是所有环境都支持，可能需要配置babel。

第3种写法：
```
class LoggingButton extends React.Component {
  handleClick() {
    console.log('this is:', this);
  }

  render() {
    // This syntax ensures `this` is bound within handleClick
    return (
      <button onClick={() => this.handleClick()}>
        Click me
      </button>
    );
  }
}
```
这个写法是将类方法的调用包装在一个箭头函数里，这算是一个trick。在onClick的位置需要一个函数引用（reference），使用的办法却是将这个函数的调用（invoke）作为一个箭头函数里的语句，然后将这个箭头函数作为引用传进去。这样就能解决所有的this的问题了，也许根本不需要推敲这种写法的运作过程，只要能得到需要的结果就行。
官网也说了，这种写法有一个性能上的弊端，就是父组件在重新渲染的时候，这个箭头函数必然会更新，而如果这个箭头函数是子组件的props的话，那么子组件也会跟着重新渲染，而这个渲染可能是没有必要的。

# useEffect能不能在render之前执行？
答案是：不能。
参考这个[博客](https://daveceddia.com/react-hook-after-render/)
useEffect只能在after render后使用，不能用在before render。useLayoutEffect同样。
如果确实需要在render之前获取数据，要么尽早return，也就意味着尽早useEffect；要么初始化一个空对象。
如果确实必须一定要在render之前执行一些操作，在父组件里执行这些操作，在子组件里条件渲染。

# 挂载后自动focus
```
import React, { useEffect, useState, useRef } from "react";
import ReactDOM from "react-dom";
function App() {
  const inputRef = useRef();
  const [value, setValue] = useState("");
  useEffect(
    () => {
      console.log("render");
      inputRef.current.focus();
    },
    [inputRef]
  );
  return (
    <input
      ref={inputRef}
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );
}
ReactDOM.render(<App />, document.querySelector("#root"));
```
> useRef()的返回值在rerender之间不会改变，所以useEffect依赖inputRef，**只在mount之后执行一次**

# useState的一些细节
```
const [inputValue, setValue] = useState("reactjs");
const [subreddit, setSubreddit] = useState(inputValue);
```
* setValue在每次渲染时是不变的，可以用作useEffect的第二个参数，只在didmount后执行一次
* inputValue是一个受控input组件的value，所以每次键入都会update并重新渲染，但是不影响第二行的useState，因为useState只使用一次初始值，会忽略后面inputValue的改变

# 把回调放在useEffect里面
```
const doStuff = () => { ... }//不要放这里
useEffect(() => {
  const doStuff = () => { ... }//放这里
  if(whatever) {
    doStuff();
  }
}, [whatever/*, doStuff*/]);//放里面就可以不用依赖doStuff
```
> 因为每次渲染的时候react都会调用组件函数，如果放在外面每次渲染都会recreate一个doStuff，如果effect依赖了doStuff则导致每次渲染都会执行effect。
> 注意：effect的依赖除了props和state，可以是任何变量包括函数

# 必须要用setState(prev=>{})的场合
```
const [list, setList] = useState([]);

const showLoading = useCallback((id, loading) => {
  // Update the list
  setList(currentList => currentList.map(item => {
    if(item.id === id) {
      return {
        ...item,
        isLoading: loading
      }
    }
    return item;
  }));
}, []); // <-- depends on nothing, now
/*如果这样写就错了，会导致无限循环
  setList(list.map(item => {
    if(item.id === id) {
      return {
        ...item,
        isLoading: loading
      }
    }
    return item;
  }));
}, [list]);
*/

useEffect(() => {
  // Connect the websocket
  const ws = new Websocket(...);

  // A message signals to reload one of the `list` items
  ws.addEventListener("message", e => {
    showLoading(e.id, true);
  });

  // Before next effect runs, close this websocket
  return () => ws.close();
}, [showLoading]); // <-- showLoading will never change
```
> 这个例子有点像做单片机通信的时候用的tcp socket，tcp的消息监听器一旦收到消息就会异步执行，包括某些setState的操作，由于是异步的setState，state的改变并不会及时反应给另一个线程
> 这个时候就需要用setState(prev=>{})以确保得到的是上一个state，否则如果直接使用局部作用域里的state，有可能state已经在别处更新了，但是这个作用域里还没有体现出来
> 另外，effect的清理函数依然在render之后才执行，和effect一样

# 至于前端要不要刷数据结构算法题什么leetcode的看法
> leetcode的中国网站带cn的那个居然打开就是一个模态登录框，我居然找不到地方关闭这个模态框，除非登录它，简直是丧心病狂吃相如狗

> 百度leetcode看到知呼上面有人问前端要不要刷题，有人回答：别人刷了你能不刷。这和辅导班别人孩子报了你能不报？是一个逻辑

> 知呼上这个问题一问，leetcode广告投放者、大厂装b者、无脑舔狗者这3者一回答，就立刻确定了答案：前端当然要刷leetcode

> 简直扯！！就学习路径和学习曲线来说，数据结构算法根本不是前端必须的，当然了作为兴趣爱好、技能完善、生涯规划，自己去刷leetcode那是自己的事，对前端来说，学习design pattern的优先级都要大于算法

> 看看[这位大神说的话](https://overreacted.io/things-i-dont-know-as-of-2018/)，人家罗列了自己不懂的技术他也不懂Algorithms，但他是react和redux核心成员。他说他知道的算法复杂度就是不要在循环里面套循环，这是一种幽默感，让生活更轻松

> leetcode是大洋彼岸文化环境下的产物，[苹果公司一半员工没有大学文凭](https://9to5mac.com/2019/03/06/tim-cook-trump-meeting-education/)，到了这里变成了一个行业内卷的文化，应试教育的延伸，机构营销的伎俩

> 因为大学生他学java他学python，就前端框架来说他学vue他学react，学校里也没有实际的项目经验，大学里只教了数据结构课程，企业只能用算法题来考察编程素养和逻辑思维了。但，就前端来说，[高中毕业后当了10年厨师的妹子也能开发前端](https://www.taniarascia.com/how-i-made-a-career-change-into-web-development/)

> [redux官网上有一段讲了useCallback和React.memo的用法](https://react-redux.js.org/api/hooks#usedispatch)，[React.memo是一个高阶函数...](https://reactjs.org/docs/react-api.html#reactmemo),就不写总结了，直接看官网文档