### 函数

#### 局部函数定义

```
local function f ()
    <函数体>
end
```

在定义递归的局部函数时，有个需要特别注意的地方，以下的方式就是错误的

```
local fact = function(n)
  if n == 0 then return 1
  else return n * fact(n-1)
  end
end
```

当lua编译到函数体中调用fact(n-1)的地方，局部的fact还没有定义完毕，因此这个表达式调用全局的fact，为了解决这个问题，可以先定义一个局部变量，然后再定义函数本身

```
local fact
fact = function(n)
  if n == 0 then return 1
  else return n * fact(n-1)
  end
end
```

### 协同程序

当一个协同程序A唤醒另一个协同程序B，协同程序A就处于一个特殊状态，既不是挂起状态（无法继续A的执行），也不是运行的状态（B在运行），这是的状态成为正常的状态



协同程序可以通过resume-yield来交换数据