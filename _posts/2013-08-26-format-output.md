---
layout: post
title:  "可变域宽度格式输出"
date:   2013-08-26 19:58:53
categories: blog
---

在 c 中， 格式转换说明中可以用 "\*" 表示宽度或精度，从而实现可变宽度的格式输出. 如：

```cpp
#include <stdio.h>

static int pretty_float(double f, unsiged int n)
{
	return printf("%0.*f\n", n, f);
}

int main(int argc, char **argv)
{
	pretty_float(3.1415926, 2);
	return 0;
}
```

而 python 除了支持这种 c 风格的变宽度指定外， 还可以这么使用：

```python
def pretty_float(f, n=3):
	return "%0.{0}f" .format(n) % f

pretty_float(3.1415926, 2)
```
也可以这样：

```python
def pretty_float_another(f, n=3):
	return "%%0.%df" %n%f

pretty_float_another(3.1415926, 2)
```

