---
title: Haskell中的Functor及类型系统
date: 2018-05-09 22:06:35
tags: [haskell, functional]
---

关于haskell中的 `Functor` 和类型系统的笔记

<!-- more -->

Functor
--------

找到一篇 [文章][Functor] 非常详细得解释了 `Functor` 以及 `Monoid` 的概念, 直接引用了

[Functor]: http://jiyinyiyong.github.io/monads-in-pictures/

```haskell
class Functor f where  
    fmap :: (a -> b) -> f a -> f b  
```

自己理解的这里的 `f` 实际上就是容器的构造函数, 也就是说对 list 来说 `f` 是 `[]`,
对 `Maybe` 来说 `f` 就是 `Maybe`

```haskell
instance Functor Maybe where  
    fmap f (Just x) = Just (f x)  
    fmap f Nothing = Nothing  
```

这里可以印证, `Just x` 和 Nothing 都是 `Maybe` 的构造函数

Kinds
--------

在 `haskell` 中所有值 (比如 1, "string", takeWhile) 都有 `type` 即类型, 
而每一种类型还有它的类型 即 __kind__

可以用 `:k` 命令来查看 `kind`

```haskell
ghci> :k Int  
Int :: *  
```

```haskell
ghci> :k Maybe  
Maybe :: * -> *  
```

```haskell
ghci> :k Maybe Int  
Maybe Int :: *  
```

其中的 `*` 符号代表一种具体类型, `Int` 是一种具体类型, 而 `Maybe` 实际上是

`Int -> Maybe Int` 也就是参数和结果都是具体类型 (Maybe是构造函数, Maybe a 才是具体类型)

```haskell
ghci> :k Either  
Either :: * -> * -> *  
```

`Either` 中 `Left` `Right` 各有一种类型, 所以它的类型参数有两个

 分析一下如下定义, t 的 `kinds` 必须是什么

```haskell
class Tofu t where  
    tofu :: j a -> t a j  
```

先从 j 开始看, 因为 `j a` 指代的是一个类型(比如 Int), 所以 j 必须是接受一个类型并生产一个类型的(比如Maybe),
因此 j 是 `* -> *` 而 a 是 `*`

因为 `t a j` 的结果也必须有是一个类型, 由此可以推导出 t 是 `* -> * -> * -> *` (`* -> (* -> *) -> *`)

再看这个例子:

```hasell
data Frank a b  = Frank {frankField :: b a} deriving (Show)  
```

Frank有两个类型参数 a, b, 由 frankField 的值可以推导出, b 为 `* -> *`, a 为 `*`
所以 Frank a b 满足 `* -> (* -> *) -> *` 这个条件

用之前的容器概念去理解的话也就是说这里的 `b` 其实是 `a` 的一个容器,
加入 a 的类型是 `Int`, b 可以是 `Maybe`, `[Int]`, 而容器又是一个具体类型,
所以可以看成 `Int -> Maybe -> Maybe Int`
或者 `Int -> [] -> [Int]`

### 最后一个例子

最后结合 `Functor` 和 `kinds` 看一个例子

```haskell
data Barry t k p = Barry { yabba :: p, dabba :: t k }  
```

如何让 `Barry` 实现 `Functor`,
可以推导出 `Berry` 的 `kind` 是 `(* -> *) -> * -> * -> *`

而 `fmap` 需要的是 `* -> *`, `Berry` 要表现成这种形式的话就要把前面的参数折叠起来,
举个例子:

```haskell
ghci> :k Barry Maybe Int                                              │
Barry Maybe Int :: * -> *
```

所以可以写成这种形式

```haskell
instance Functor (Barry a b) where
fmap f (Barry {yabba = x, dabba = y}) = Barry {yabba = f x, dabba = y}  
```

这里的 f 就是 `Barry a b` (由于柯里化), 所以我们容器里面待转换的值其实是 p,
因此 `yabba` 中的 `p` 值需要用被转换, 而 `dabba` 中的不需要 
