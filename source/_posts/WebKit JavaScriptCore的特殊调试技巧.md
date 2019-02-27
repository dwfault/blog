---
title: WebKit JavaScriptCore的特殊调试技巧
---

### 一、jsc内置函数

jsc作为WebKit的JavaScriptCore的一个独立的可执行实体，其内置了一些特殊函数来帮助调试。这些函数位于JavaScriptCore/jsc.cpp，在finishCreation函数中通过addFunction函数注册：

```c++
        addFunction(vm, "debug", functionDebug, 1);
        addFunction(vm, "describe", functionDescribe, 1);
        addFunction(vm, "describeArray", functionDescribeArray, 1);
        addFunction(vm, "print", functionPrintStdOut, 1);
        addFunction(vm, "printErr", functionPrintStdErr, 1);
        addFunction(vm, "quit", functionQuit, 0);
        addFunction(vm, "gc", functionGCAndSweep, 0);
        addFunction(vm, "fullGC", functionFullGC, 0);
        addFunction(vm, "edenGC", functionEdenGC, 0);
        addFunction(vm, "forceGCSlowPaths", functionForceGCSlowPaths, 0);
        addFunction(vm, "gcHeapSize", functionHeapSize, 0);
        ...
        addFunction(vm, "addressOf", functionAddressOf, 1);
        addFunction(vm, "run", functionRun, 1);
        addFunction(vm, "runString", functionRunString, 1);
        addFunction(vm, "load", functionLoad, 1);
        addFunction(vm, "loadString", functionLoadString, 1);
        addFunction(vm, "readFile", functionReadFile, 2);
        addFunction(vm, "read", functionReadFile, 2);
        ...
        addFunction(vm, "jscStack", functionJSCStack, 1);
        addFunction(vm, "readline", functionReadline, 0);
        ...
        addFunction(vm, "neverInlineFunction", functionNeverInlineFunction, 1);
        addFunction(vm, "noInline", functionNeverInlineFunction, 1);
        addFunction(vm, "noDFG", functionNoDFG, 1);
        addFunction(vm, "noFTL", functionNoFTL, 1);
        addFunction(vm, "noOSRExitFuzzing", functionNoOSRExitFuzzing, 1);
        addFunction(vm, "numberOfDFGCompiles", functionNumberOfDFGCompiles, 1);
        addFunction(vm, "jscOptions", functionJSCOptions, 0);
        addFunction(vm, "optimizeNextInvocation", functionOptimizeNextInvocation, 1);
        addFunction(vm, "reoptimizationRetryCount", functionReoptimizationRetryCount, 1);
        addFunction(vm, "transferArrayBuffer", functionTransferArrayBuffer, 1);
        addFunction(vm, "failNextNewCodeBlock", functionFailNextNewCodeBlock, 1);
```

这些函数不是ECMAScript标准涉及的函数，一般的JavaScript引擎实现中也不包含这些函数。可以认为这些函数是为了方便开发者而置入的。例如print函数用来打印变量到控制台，在nodejs等环境中对应console.log。这些函数很有用处，可以大大简化调试。例如函数describe、describeArray可以透露很多信息：

```javascript
let split = '------';

let a = [1,2];

print(describe(a));
print(split);
print(describeArray(a));
print(split);
print(addressOf(a));
print('0x' + f64_to_uint(addressOf(a)).toString(16));

print(split);
a.length = 3;

print(describe(a));
print(split);
print(describeArray(a));
print(split);
print(addressOf(a));
print('0x' + f64_to_uint(addressOf(a)).toString(16));

function f64_to_uint(f64) {
    var bytes = new Uint8Array(new Float64Array([f64]).buffer);
    if (bytes.length != 8) {
        if (debug)
            print("f64_to_u64 error.");
    }
    var uint = 0;
    for (var i = 0; i < bytes.length; i++) {
        uint += (bytes[bytes.length - i - 1] * Math.pow(2, 0x38 - i * 8));
    }
    return uint
}
```

得到如下内容，从中甚至可以看到对象的transition变化：

```c
Object: 0x62d0000a4340 with butterfly 0x62d0001a8010 (Structure 0x62d00000ec30:[Array, {}, CopyOnWriteArrayWithInt32, Proto:0x62d00007c0a0, Leaf]), StructureID: 102
------
<Butterfly: 0x62d0001a8010; public length: 2; vector length: 2>
------
5.3678005860554e-310
0x62d0000a4340
------
Object: 0x62d0000a4340 with butterfly 0x62d000220008 (Structure 0x62d00000ea00:[Array, {}, ArrayWithInt32, Proto:0x62d00007c0a0, Leaf]), StructureID: 97
------
<Butterfly: 0x62d000220008; public length: 3; vector length: 3>
------
5.3678005860554e-310
0x62d0000a4340
```

也可以获得JS的执行栈：

```javascript
function f(){
    f64_to_uint(5.3678005860554e-310 + parseInt(jscStack()));
}
f();
```

得到：

```javascript
--> Stack trace:
    0   jscStack@[native code]
    1   f@yuebao.js:36:57
    2   global code@yuebao.js:39:2
```

另外，noDFG、noFTL等函数规定了特定js函数在JIT时的特定行为；gc、fullGC、edenGC可以控制垃圾回收等。由于依靠这些函数的漏洞触发不会被承认，所以这些函数看起来比较鸡肋，还是要手动触发jit、gc比较符合常理。因此这些函数在jsc中的高阶用法在这里不再讨论。

阅读这些函数可以理解很多基本类型的用法和关系，例如JSValue、jsNumber、jsString、Butterfly、Array、WTF::Vector。



### 二、使用Arrary.prototype.slice下断点

类似于IE漏洞分析中的tan等函数，Array.prototype.slice也可以被用作断点。目标函数位于JavaScriptCore/runtime/ArrayPrototype.cpp：

```javascript
b arrayProtoFuncSlice
```

在js中直接引用：

```javascript
Array.prototype.slice([]);
```

断下后会停在该函数的开头。



### 三、自定义dbg()函数下断点

可以使用x86 int3指令自定义一个断点函数dbg()进行调试。在jsc中自定义函数只需要三个步骤：

- 声明函数

- 定义函数
- 注册函数
  - addFunction的第二个参数是对应的js函数的名字，第四个参数是js函数的参数的个数。

![image-20190103172422856](WebKit JavaScriptCore的特殊调试技巧.assets/image-20190103172422856-6507462.png)

自定义的Native函数参数统一为ExecState类型，这是一个为了确定上下文的类型。在使用addFunction注册函数时，第二个参数是该函数参数个数，这些参数可以通过ExecState类型的exec参数取得，因此dbg()函数还可以定义为有参数的函数：

```c++
static int dbgBrCnt = 0;
EncodedJSValue JSC_HOST_CALL functionDbg(ExecState* exec){
    auto times = exec->uncheckedArgument(0).toInteger(exec);
    if(dbgBrCnt < times){}
    else
        asm("int3;");
    dbgBrCnt++;
    return JSValue::encode(jsUndefined());
}
```

因此实现了一个条件断点，这样可以让函数在循环到某确定次数的时候陷入int3，比如下面定义的是一个到达1000次之后触发的断点函数：

```javascript
dbg(1000);
```

当然上述定义的断点适合简单的情况，只有一个断点的时候是可运行的。

functionDbg()的内容还可以更多样化地进行定制。这在调试JIT、垃圾回收等等复杂情况的时候尤其有用。



### 四、为dbg()函数适配DFG JIT

上述所有内容都是JSC_HOST_CALL类型的静态函数，在JIT当中使用这些函数会不可避免地使栈回溯发生变化，因为存在一个从JITed代码跳转出来的过程；有时候还会为DFG JIT添加OSRExit的退出点，使得本来应该触发的漏洞在dbg()函数提前回退，导致漏洞无法触发。

JavaScriptCore的运行是一个比较复杂的过程，可以参考《MOSEC2018分享》中的内容，程序运行的上下文会经常在Baseline JIT和DFG JIT生成的代码之间切换。于是上文中的functionDbg()函数：

- 基本不会影响到Baseline JIT代码生成，只是相当于在一段直线式程序中添加结点。
  - 调试时，正常通过该结点即可。
- 会极大地影响DFG JIT代码生成。
  - 由DirectCall(DFG IR)调用，从JITed代码跳转出来
    - 后跟InvalidationPoint(DFG IR)结点
    - 后跟CheckStructure(DFG IR)结点

因此，dbg()函数可以保证插入点之前的代码的执行情况是与不插入的情况吻合的，但其后的代码执行逻辑可能会发生改变。例如，在JavaScriptCore JIT系统中最典型的类型混淆漏洞，其中有：

```javascript
前文
类型混淆的赋值
后文
```

为了观察类型混淆漏洞，会在“类型混淆的赋值”之前插入一个DirectCall结点调用dbg()：

```javascript
前文
DirectCall
CheckStructure
InvalidationPoint
类型混淆的赋值
后文
```

于是在CheckStructure之后程序就进行了OSRExit，无法到达“类型混淆的赋值”，因此发生所谓观察者效应，调试失败。

为了解决这个问题，可以把CheckStructure这一DFG IR的编译过程注释掉，这样就可以随心所欲地下断点了：

JavaScriptCore/dfg/DFGSpeculativeJIT64.cpp

```c++
SpeculativeJIT::compile(Node * node)
    case CheckStructure:{
        //compileCheckStructure(node);
        break;
}
```

当然，这样的情况不适用CheckStructure本身出问题的情况。



在其他的情况下，为了去掉InvalidationPoint还通过可以修改DirectCall这一DFG IR的读写属性标识，将其指明的Side Effect去掉：

JavaScriptCore/dfg/DFGClobberize.h

```javascript
switch (node->op()){
	...
    case DirectCall:
        //read(World);
        read(Heap);
        //write(Heap);
        write(SideState);
        return;
}
```



### 五、DFG JIT代码断点

更直观的方法，是在DFG代码生成的位置下断点：

![image-20190103172811046](WebKit JavaScriptCore的特殊调试技巧.assets/image-20190103172811046-6507691.png)

需要参数：

```bash
--dumpDFGDisassembly=true
```

配合内存dump+反汇编器分析更容易。