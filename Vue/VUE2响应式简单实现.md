## VUE2响应式简单实现

### reactive.js 相当于Observe

~~~javascript
import Dep from './dep';

// 简单实现 默认传入的都是对象(假装typeof 出来的都是对象 没有数组)
export function reactive(data){
    if(typeof data === 'object'){
        Object.keys(data).forEach(key => {
            defineReactive(data,key)
        })
    }
    return data;
}

// 定义响应式
function defineReactive(data,key){
    let val = data[key];

    const dep = new Dep();
    Object.defineProperty(data,key,{
        get(){
            dep.depend();
            return val
        },
        set(newVal){
            if(newVal === val) return;
            val = newVal;
            dep.notify();
        }
    })

    if(typeof val === 'object'){
        reactive(val)
    }
}
~~~
### dep.js //收集watcher，通知watcher更新

~~~javascript
export default class Dep {
    constructor(){
        // 收集watcher
        this.subs = [];
    }

    addSub(watcher){
        this.subs = this.subs.filter(item => item !== watcher);
        this.subs.push(watcher);
    }

    // 收集依赖
    depend(){
        if(Dep.target){
            Dep.target.addDep(this)
        }
    }

    // 通知依赖数据变化
    notify(){
        this.subs.forEach(watcher => {
            watcher.update();
        })
    }
}

Dep.target = null;

// watcher栈,比如父子组件嵌套时，会把父组件watcher放入栈中
const targetStack = [];

export function pushTarget(target){
    if(Dep.target){
        targetStack.push(Dep.target)
    }
    Dep.target = target;
}

export function popTarget(){
    Dep.target = targetStack.pop()
}
~~~


### watcher.js //主要用来触发函数调用

~~~javascript
import Dep, { pushTarget, popTarget} from "./dep";
// 组件的渲染函数,computed, watch都会生成watcher
export default class Watcher{
    constructor(getter,options = {}){
        const { lazy, watch, callback } = options;

        this.getter = getter; //渲染函数 
        this.deps = [];

        this.lazy = lazy;
        this.dirty = lazy; //true时,取值时重新运行getter,false时直接用watcher的value

        this.watch = watch;
        this.callback = callback; //watch的回调函数

        if(!lazy){ //如果不是computed的直接运行get
            this.get();
        }
    }

    // 运行渲染函数
    get(){
        pushTarget(this); //Dep.target设置成当前watcher实例
        this.value = this.getter(); //运行渲染函数，Dep会收集依赖（存入watcher）
        popTarget();
        return this.value;
    }

    addDep(dep){
        // watcher储存dep
        this.deps = this.deps.filter(item => item !== dep);
        this.deps.push(dep);
        // dep储存watcher
        dep.addSub(this)
    }
    // 触发computed的get时，Dep.target变为computed的watcher,dep收集这个watcher
    // dep触发notify时，会运行这个watcher的update,把dirty设为true
    // 当有渲染函数再次触发computed的get，则会运行evaluate,重新运行get
    depend(){
        this.deps.forEach(dep => dep.depend());
    }
    // computed从新获取值
    evaluate(){
        this.value = this.get();
        this.dirty = false; //false时等待取值时执行返回this.value
    }

    // 数据更新后，运行
    update(){
        if(this.lazy){
            this.dirty = true; //计算属性的更新，直接把dirty改为true,下次取值重新运行get
        }
        else if(this.watch){
            const oldValue = this.value;
            this.get();
            this.callback(this.value,oldValue)
        }
        else {
            this.get();
        }
    }
}
~~~

### computed.js

~~~javascript
import Dep from "./dep";
import Watcher from "./watcher";

// 惰性取值
export default function computed(getter){
    let def = {};
    const computedWatcher = new Watcher(getter,{ 
        lazy: true, //生命lazy属性，标记 computed wathcer
    });
    Object.defineProperty(def,'value',{
        get(){
            // 重新执行getter函数,主要判断是不是需要从缓存读取数据
            if(computedWatcher.dirty){
                computedWatcher.evaluate();
            }

            if(Dep.target){
                computedWatcher.depend(); //把当前渲染函数的watcher收集到computedWatcher的dep中
            }
            
            return computedWatcher.value; //返回getter运行后的值
        }
    })

    return def; //使用的时候用 .value取值
}
~~~

### watch.js

~~~javascript
import Watcher from "./watcher";
export default function watch(getter,callback){
    new Watcher(getter,{
        watch: true,
        callback
    })
}
~~~ 

### 使用举例

~~~html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <div>
        <span>名称:</span> <span id="app"></span>
    </div>
    <div>
        <span>数字:</span> <span id="num"></span>
    </div>
    <div>
        <span>computed:</span> <span id="computed"></span>
    </div>
</body>
</html>
~~~

~~~javascript
import { reactive } from './reactive';
import Watcher from './watcher';
import computed from './computed';
import watch from './watch';

window.data = reactive({
    num: 1,
    detail: {
        name: '响应式原理'
    }
})

const computedData = computed(() => data.num * 2);

watch(
    () => data.num,
    (newValue,oldValue) => {
        console.log(newValue,oldValue)
    }
)

new Watcher(() => {
    document.getElementById('app').innerHTML = data.detail.name;
    document.getElementById('num').innerHTML = data.num;
    document.getElementById('computed').innerHTML = computedData.value;
})
~~~
