# Vue 3.0 Composition API

## setUp() 函数

这个函数将会是我们 setup 我们组件逻辑的地方，它会在一个组件实例被创建时，
初始化了 props 之后调用。setup() 会接收到初始的 props 作为参数

```javascript
import { ref } from 'vue'

const MyComponent = {
  setup(props) {
    const msg = ref('hello')
    const appendName = () => {
      msg.value = `hello ${props.name}`
    }
    return {
      msg,
      appendName
    }
  },
  template: `<div @click="appendName">{{ msg }}</div>`
}
```

## ref() 函数

返回一个包装对象，`{ value: String | Number}`

### 为什么需要包装对象？

> 包装对象的意义就在于提供一个让我们能够在函数之间以引用的方式传递任意类型值的容器。
> 这有点像 React Hooks 中的 useRef —— 但不同的是 Vue 的包装对象同时还是响应式的数据源。
> 有了这样的容器，我们就可以在封装了逻辑的组合函数中将状态以引用的方式传回给组件。
> 组件负责展示（追踪依赖），组合函数负责管理状态（触发更新）

```javascript
export default {
  setup() {
    const valueA = useLogicA(); // valueA 可能被 useLogicA() 内部的代码修改从而触发更新
    const valueB = useLogicB();
    return {
      valueA,
      valueB
    };
  }
};
```

### 包装对象的自动展开

当包装对象被暴露给模版渲染上下文，或是被嵌套在另一个响应式对象中的时候，
它会被自动展开 (unwrap) 为内部的值

#### 渲染上下文时

```javascript
export default {
  setup() {
    return {
      count: ref(0)
    };
  },
  template: `<button @click="count++">{{ count }}</button>`
};
```

#### 被响应式对象的属性引用时

```javascript
const count = ref(0);
const obj = reactive({
  count
});

console.log(obj.count); // 0

obj.count++;
console.log(obj.count); // 1
console.log(count.value); // 1
```

## computed Value (计算值)

计算值的行为跟计算属性 (computed property) 一样：只有当依赖变化的时候它才会被重新计算。
`computed()` 返回的是一个只读的包装对象

```javascript
const count = ref(0);
const countPlusOne = computed(() => count.value + 1);
```

## Watchers

### watch()

`watch()` API 提供了基于观察状态的变化来执行副作用的能力。
`watch()` 接收的第一个参数被称作「 数据源 」，它可以是：

- 一个返回任意值的函数
- 一个包装对象
- 一个包含上述两种数据源的数组

第二个参数是回调函数。回调函数只有当数据源发生变动时才会被触发

不同:

- `watch()` 的回调会在创建时就执行一次
- `watch()` 的回调在触发时，DOM 总是会在一个已经被更新过的状态下

### 观察 props

setup() 接收到的 props 对象是一个可观测的响应式对象

```javascript
const MyComponent = {
  props: {
    id: Number
  },
  setup(props) {
    const data = ref(null);
    watch(
      () => props.id,
      async id => {
        data.value = await fetchData(id);
      }
    );
    return {
      data
    };
  }
};
```

### 观察包装对象

```javascript
const count = ref(1);
// double 是一个计算包装对象
const double = computed(() => count.value * 2);

watch(double, value => {
  console.log("double the count is: ", value);
}); // -> double the count is: 0

count.value++; // -> double the count is: 2
```

### 观察多个数据源

```javascript
watch([refA, () => refB.value], ([a, b], [prevA, prevB]) => {
  console.log(`a is: ${a}`);
  console.log(`b is: ${b}`);
});
```

### 停止观察

`watch()` 返回一个停止观察的函数

```javascript
const stop = watch();
// stop watching
stop();
```

### 清理副作用

有时候当观察的数据源变化后，我们可能需要对之前所执行的副作用进行清理。
为了处理这种情况，watcher 的回调函数会接收到的第三个参数是一个用来注册清理操作的函数.

- 在回调被下一次调用前
- 在 watcher 被停止前

```javascript
watch(idValue, (id, oldId, onCleanup) => {
  const token = performAsyncOperation(id);
  onCleanup(() => {
    // id 发生了变化，或是 watcher 即将被停止.
    // 取消还未完成的异步操作。
    token.cancel();
  });
});
```

### 回调的调用时机

默认情况下，所有的 watcher 回调都会在当前的 renderer flush 之后被调用。
如果你想要让回调在 DOM 更新之前或是被同步触发，可以使用 flush 选项

```javascript
watch(
  () => count.value + 1,
  () => console.log(`count changed`),
  {
    flush: "post", // default, fire after renderer flush
    flush: "pre", // fire right before renderer flush
    flush: "sync" // fire synchronously
  }
);
```

#### 全部的 watch 选项

```typescript
interface WatchOptions {
  lazy?: boolean;
  deep?: boolean;
  flush?: "pre" | "post" | "sync";
  onTrack?: (e: DebuggerEvent) => void;
  onTrigger?: (e: DebuggerEvent) => void;
}

interface DebuggerEvent {
  effect: ReactiveEffect;
  target: any;
  key: string | symbol | undefined;
  type: "set" | "add" | "delete" | "clear" | "get" | "has" | "iterate";
}
```

## watchEffect() 函数

当内部的依赖变化时，就会重新执行
eg： 当内部的 `count` 变化

```javascript
export default {
  setup() {
    const count = ref(0);

    watchEffect(() => {
      console.log(count.value);
    });

    return {
      count
    };
  }
};
```

## readonly() 函数

参数: `reactive` `ref` `plain` 对象
返回值: 只读的对象引用

任何的嵌套属性都是只读的

```javascript
const original = reactive({ count: 0 });

const copy = readonly(original);

watchEffect(() => {
  // works for reactivity tracking
  console.log(copy.count);
});

// mutating original will trigger watchers relying on the copy
original.count++;

// mutating the copy will fail and result in a warning
copy.count++; // warning!
```

## 生命周期函数

所有现有的生命周期钩子都会有对应的 onXXX 函数（只能在 setup() 中使用）

```javascript
import { ,onMounted, onUpdated, onUnmounted } from "vue";

const MyComponent = {
  setup() {
    onMounted(() => {
      console.log("mounted!");
    });
    onUpdated(() => {
      console.log("updated!");
    });
    // destroyed 调整为 unmounted
    onUnmounted(() => {
      console.log("unmounted!");
    });
  }
};
```
