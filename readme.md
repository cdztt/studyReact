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