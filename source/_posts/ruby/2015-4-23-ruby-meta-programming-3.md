---
layout: post
title: Ruby元编程系列3 – 作用域
date: 2015-04-23
categories: Programming
tags: [Ruby, Meta Programming]
---

## 代码块和闭包

代码块是Ruby语法的一大特色，可以通过两种方式来定义代码块：

* 将代码块定义在大括号中，通常只有一行代码的块使用这种方式。
* 将代码定义在do…end 中，通常具有多行代码的块使用这种方式。


<!--more-->

定义接受代码块的函数也有两种方式：

* 通过yield关键字
* 给函数的最后一个参数加上&前缀，来接受代码块

分别看对应的例子：


```ruby
#!/usr/bin/ruby

def code_block_sample_1
    puts "hello " + yield if block_given?
end

code_block_sample_1 {"world"} # => "hello world"

def code_block_sample_2(&block)
    puts "hello " + block.call if block 
end

code_block_sample_2 {"yue"} # => "hello yue"
```

关于闭包，以下是维基的定义：Operationally, a closure is a data structure storing a function together with an environment. 具体到Ruby中，就是指代码块与其运行环境的绑定。

一个简单的示例：

```ruby
#!/usr/bin/ruby

name = "yue"

3.times {puts name}
```

可以看出由于闭包的存在，代码块可以获取到局部变量name.

## 作用域

在Java和C#中，有内部作用域(inner scope)的概念，内部作用域可以看到外部作用域中的变量。但是Ruby中没有这种嵌套式的作用域，它的作用域之间是泾渭分明的。

Ruby作用域有三重门——有三个地方会打开新的作用域：

* 类定义
* 模块定义
* 方法定义

但是很多时候，你在定义类、模块、方法的时候，希望仍然能够访问之前局部作用域中的变量，不希望创建新的作用域。我们可以使用一种被称作“扁平作用域”(Flat Scope)的方式来穿越作用域门。

代码示例：

```ruby
#!/usr/bin/ruby

name = "yue"

MyClass = Class.new do 
    define_method :my_method do
       puts name 
    end
end

MyClass.new.my_method # => "yue"
```
可以看到，通过调用Class.new和define_method, name变量穿越了类定义和方法定义这两重作用域门。

## class_eval()与instance_eval()

class_eval() 与instance_eval()是Ruby元编程中常见的两种方法调用，在调用的过程中都用到了扁平作用域的技术。

class_eval()的接受者只能是类，可以将类名当作变量传递，相当于增加了一层间接性。

```ruby
#!/usr/bin/ruby

class Sample_1
    
end

class Sample_2
    
end

def add_method_to_class(cls)
    cls.class_eval  do 
        def greet
           puts "hello" 
        end
    end
end

add_method_to_class(Sample_1)
add_method_to_class(Sample_2)

Sample_1.new.greet
Sample_2.new.greet
```

instance_eval()的调用则略显微妙。

当以一个类作为调用者时，是把这个类当作一个instance, 于是当前对象(receiver)仍然是这个类，但是当前类变成这个类的类——Class 类。此时若在代码块中定义实例方法，效果相当于在定义该类的类方法（具体细节会在下一篇讲singleton class 时详细解释）。

代码示例：

```ruby
class Nerd

end

Nerd.instance_eval do 
    def greet
        puts "hello" 
    end
end

Nerd.greet # => "hello"
```

当以一个实例来调用instance_eval()时，和通常的调用实例方法没太大不同，最重要的区别是instance_eval()可以打破封装，可以访问到对象的私有变量。这种技术被称为“上下文探针”，在单元测试的时候非常有用。

代码示例：

```ruby
#!/usr/bin/ruby

class Nerd
    def initialize
        @name = "yue"
    end
end

Nerd.new.instance_eval do 
   puts @name # => "yue"
end
```

关于作用域的知识我们就介绍到这里，下一篇会介绍单件方法和模块。

