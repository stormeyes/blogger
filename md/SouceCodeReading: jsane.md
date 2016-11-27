### Github拾遗 jsane源码走读

在Python中访问一个字典可以用dict_obj['keyname']来访问, 而通过dict_obj.keyname访问会报错

```python
AttributeError: 'dict' object has no attribute 'a'
```

为什么Python的字典没有内置使用点号访问属性的操作呢? 很简单, 在Python中点号操作意味着是访问类/对象 的attr, 而dict作为内置的数据类型而不是类/对象使用下标点访问属性, 一方面违反了Python一致性&&明确性的设计原则的, 另一方面, 需要从Python的实现说起. 

在js中, key的类型必须是string, 这是因为js中所有对象都是通过哈希实现的, 强制的string类型使得开发者可以放心地在对象中使用链式调用. 在Python中, 字典的key可以是除了列表以外的任何东西(严格上来说也不是这样), 比如字符串, 数字, 甚至是函数, 异常, 元祖, 对象等一切提供了**\_\_hash\_\_**函数的对象(user-defined classes are hashable by default) , 这种情况下使用下标点访问会让解释器困惑. (关于为什么列表不能作为key, Python的[官方文档](https://wiki.python.org/moin/DictionaryKeys)有详细的解释)

如果强制想要使用下标点访问属性, 可以通过以下两种方式:

```python
from collection import namedtuple

User = namedtuple('User', ['age', 'name', 'gender'])
user = User(11, 'kongkongyzt', 'man')

# 通过不可变的namedtuple锁死变量, 适用于运行过程中不会再发生变动, 只读的数据结构, 如配置参数, const实现
print(user.age, user.name, user.gender)
```

```python
class dot_dict(dict):
    def __init__(self, dict_object):
        self.dict_object = dict_object
        
    def __getattr__(self, key):
        return self.dict_object[key]

"""
在访问变量的下标点的时候python不会关心变量的类型是否允许这种形式的调用, 只要实现了__getattr__方法即可,这是动态语言 duck typing 特性的体现
"""
user = dot_dict({'name': 'kongkongyzt'})
print(user.name)
```

上面的实现都是demo级别的, 比较粗糙, 比如a.b.c.d这种链式调用的情况就没处理. Github上有人比较完整地实现了这个功能, 叫 [jsane](https://github.com/skorokithakis/jsane)

jsane主要完善了以下问题:

+   实现链式调用的同时, 避免了传统字典需要逐层检查数据是否异常的问题:

    ```python
    root = my_json.get("root")
    if root is None:
        return None

    key1 = root.get("key1")
    if key1 is None:
        return None

    key2 = key1.get("key2")
    if key2 is None:
        return None

    <five more times>
    ```

+   当找不到数据的时候, 可以指定返回默认的数据:

    ```python
    import jsane

    j = jsane.loads('{"some": {"json": "b"}}')
    j.some.c(default=3)
    ```

老实说我觉得jsane的源码写得不怎么样, 不过因为接口少, 代码少, 结构清晰, 作为一个Python初学者阅读的第一份开源代码还是很合适的.

jsane源码只有三个文件, ``__init__.py``, ``traversable.py``,``wrapper.py``

#### \__init__\.py

代码很短, 可以直接放上来:

```python
# flake8: noqa
from .traversable import JSaneException
from .wrapper import load, loads, dump, dumps, from_dict

__version__ = '0.1.0'
```

这里主要是将供外部调用的接口暴露出来, 挂载在jsane模块下面作为一级调用接口, 符合大多数Python第三方模块对init文件的做法. load, loads, dump, dumps和Python标准库提供的接口一模一样, 分别是json数据的导入和导出. 这个from_dict字面上的意思应该是将一个dict对象转换成类json的封装对象, 在控制台试了一下:

```python
import jsane

j = jsane.from_dict({'a': 'b', 'c': 'd'})
print(j) # <Traversable: {'c': 'd', 'a': 'b'}>
```

#### wrapper.py

代码也很短, 直接贴代码:

```python
import json

from .traversable import Traversable


def load(*args, **kwargs):
    j = json.load(*args, **kwargs)
    return Traversable(j)


def loads(*args, **kwargs):
    j = json.loads(*args, **kwargs)
    return Traversable(j)


def dump(*args, **kwargs):
    return json.dump(*args, **kwargs)


def dumps(*args, **kwargs):
    return json.dumps(*args, **kwargs)


def from_dict(jdict):
    return Traversable(jdict)
```

load和loads就是借助Python标准库的json接口, 将json和dict都统一转换为dict后, 返回Traversable对象, 为了保持接口的一致性, 顺带实现了dump和dumps, 全是对接口的封装, 名副其实的wrapper

#### traversable.py

这个的代码也很短, 贴出来:

```python
# Use this as a detectable default value.
DEFAULT = object()


class JSaneException(Exception):
    pass


class Empty(object):
    def __init__(self, key_name=""):
        self._key_name = key_name

    def __getattr__(self, key):
        return self
    __getitem__ = __getattr__

    def __repr__(self):
        return "<Traversable key does not exist: %s>" % self._key_name

    def __call__(self, default=DEFAULT):
        if default is DEFAULT:
            raise JSaneException("Key does not exist: %s" % self._key_name)
        else:
            return default


class Traversable(object):
    def __init__(self, obj):
        self._obj = obj

    def __getattr__(self, key):
        try:
            return Traversable(self._obj[key])
        except (KeyError, AttributeError, IndexError, TypeError):
            return Empty(key)
    __getitem__ = __getattr__

    def __eq__(self, other):
        "Equality test."
        if not hasattr(other, "_obj"):
            return False
        return self._obj == other._obj

    def __repr__(self):
        return "<Traversable: %r>" % self._obj

    def __call__(self, default=DEFAULT):
        """
        Resolve the object.

        This will always succeed, since, if a lookup fails, an Empty instance
        will be returned farther upstream.
        """
        return self._obj
```

无论是从文件拿到的json, 还是字符串json, 还是字典都会被转换成Traversable对象, 该对象通过下标点访问属性的时候, 解释器会通过调用\_\_getattr__ 拿到值, 因此Traversable中的\_\_getattr__先拿到值value, 又返回了用该值构造的Traversable对象, 后面的节点又通过下表点调用新构造的Traversable对象的\_\_getattr\_\_ 方法, 循环往复, 循环从而达到链式调用的目的.

拿到值的过程会有几种异常:

+   键异常, 键由特殊符号等组成, 不符合Python中键的规范
+   属性异常, 一般是属性不存在, 比如访问了不存在的键值对
+   索引异常, 元素可能是一个数组, 有可能触发数据越界
+   类型异常, 比如元素是数组, 本应使用数字访问却传入了类似"aaa"的字符串, 会导致TypeError发生

这几种异常的情况下, jsane会返回一个代表空的对象, 对这个空对象的访问会再次返回它本身(也就是一个空对象), 这样就可以避免每次使用调用链都要try... catch... 造成的麻烦, 如果调用链中产生了异常值, 可以指定一个默认的返回值(通过_call__实现, 下文有描述)

Empty类和Traversable类虽然有相似之处, 但是对关键的\__getattr__方法的实现是不一样的, 因此还是拆开来比较合适.

将\_\_getitem\_\_ 修改为与自定义的 \__getattr__方法一样, 目的是处理以下调用链混用的情况:

```json
b.a['c'].d.e['f'].g()
```



调用链每个调用节点返回的是一个Traversable对象, 但是作为使用者我们希望调用结束后最终拿到的数据的类型就是str/int/float/bool/dict 等等基本类型, 因此, 对于调用链的最后一个一个节点返回的对象要执行一次call.

Python的确是一门很神奇的语言, Python的语言特性允许对象是callable的, 只要实现了\_\_call__ 方法即可, 因此只需要定义 \_\_call__ 方法并返回value即可.

```python
class test_obj():
    def __call__(self):
        print('obj been called')
        
T = test_obj()
T()
```

作者比较细心的是为每个类都定义了\_\_repr\_\_方法, 通过print可以明确地看出返回的数据的类型是Traversable, 方便调用者发现问题.

当到达最后一个调用节点, 执行对象的call并返回最终值的时候, 这个对象要么是Traversable要么是Empty, Traversable对象的\_\_call\_\_方法其实不需要default参数, 因为Traversable对象一定是有value的, Empty对象才需要一个默认值, 因此在Traversable的\_\_call\_\_方法定义default参数没有任何意义, 这个参数永远不会被使用, 但是依旧要定义这个参数, 因为Python中函数/方法的参数数量不匹配会引发TypeError错误, 如果在一个正常的无空值调用链上定义了default, call的时候就会因为方法参数数量不一致抛出异常.