---
available_keys: title, category, description, date, ref, author, tags: List
title: Python: 相对引用和绝对引用，哪个更好?
tags: Python, 工程化
---
我认为在submodule中使用相对引用是合适的，但在一次提交merge request时，被一个曾任职于Google的权威的工程师拒绝了，原因是：[Google Python Style Guide](https://google.github.io/styleguide/pyguide.html) 禁止使用相对引用：
> Do not use relative names in imports. Even if the module is in the same package, use the full package name. This helps prevent unintentionally importing a package twice.

Python的两种引用方式，其中相对引用就是以当前代码文件所在位置为参考点，引用其他模块，如：
```python
from . import xxx
from ...xxx import yyy
```
而绝对引用，则是以入口文件或者顶级模块的位置作为参考点，引入其他模块：
```python
from xxx import yyy
from xxx.yyy import zzz
```
有些IDE把这个入口点称为source root，其```__name__```值为```"__main__"```。当我们要联合两个git仓库的代码时，要使用绝对引用的方式调用submodule里的python代码，有两种途径：
* 使main repo和submodule的source root保持一致
* 在main repo引用submodule之前，加入一句```import sys;sys.path.append({source_root_of_submodule})```

第一种方法很容易理解，Python的引用机制确保了它能够搜索到source root下的模块；而当source root不一致时，会得到一个ModuleNotFoundError，因为import参考点不一样，解释器就找不到模块的正确位置。这意味着，只能把submodule放在main repo的source root下。当submodule一多，main的source root会变得臃肿不堪。更加致命的是，如果submodule的入口点在它的子目录下（很常见，很多仓库最外层是一些licence、document等，代码放在一个叫src的子目录里），在main repo里就无法import它，因为这种代码结构注定了submodule的source root比main repo至少深一个层级。

第二种方式是将submodule的source root加入到main repo的sys.path中。Python解释器在搜索不到模块时，会尝试在sys.path的对应位置查找，所以这种方式看起来是可以work的。但是，```sys.path.append```是一个非常危险的操作！submodule里的包会覆盖sys.path里的同名包，可能会对sys.path造成污染，整个过程都将是悄无声息的。运气不好的话，程序开始启动时能正常运行，但后续会在某个隐藏条件被触发时，突然的产生致命bug。

以上的情形中，使用绝对引用无法避免这些问题。更优雅的解决办法是，submodule里除了main.py、test.py等这些入口文件之外，其他全部使用相对引用。这样main repo可以根据自己的代码结构，灵活地把submodule放在任意位置，此时submodule的source root取决于main的source root，不会再有冲突。

其实不仅仅是google官方的pyguide不建议使用相对引用，诸多大厂的python规范都是如此，Python官方也曾一度要去除相对引用的方式，pep328还介绍了两方阵营的争论过程，但最终相对引用还是得以保留。不用相对引用的原因很多，如pyguide所述"防止包被引用两次"，还有它使目录结构不够清晰了然、给人容易出错的感觉等，但不使用绝对引用的理由只有一个，就是在多个代码仓库合并开发时，如果不借助pypi管理包的话，就无法统一参考点。代码规范并非在任何情况下都是金科玉律，在实践中永远要选择合适的方式。

[https://docs.python.org/3.11/tutorial/modules.html#intra-package-references](https://docs.python.org/3.11/tutorial/modules.html#intra-package-references)
[https://peps.python.org/pep-0328/](https://peps.python.org/pep-0328/)