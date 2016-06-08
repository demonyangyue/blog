---
layout: post
title: Ruby元编程系列1 -- 对象和类
date: 2015-04-14
categories: Programming
tags: [Ruby, Meta Programming]
---

## 深入Ruby 对象

在Ruby中，对象由一组实例变量和一个指向其类型的引用组成。

<!--more-->

```ruby
class Nerd
    attr_reader :name
    def initialize
       @name = "yue" 
       @age = 25
    end
end

nerd_1= Nerd.new()
nerd_2= Nerd.new()

nerd_1.name.equal?(nerd_2.name)   # => false
nerd_1.class.equal?(nerd_2.class) # => true
```

可以看出，nerd_1 和 nerd_2 拥有独立的实例变量，但引用了相同的类。

Ruby 的对象模型和其它面向对象的编程语言类似：

![](/images/ruby_object_model.jpg)
与Java这样的静态语言不一样，ruby对象的实例变量可以在运行过程中动态添加：

```ruby
class Nerd
   
    def make_stupid
        @stupid = true 
    end
end

nerd_1.make_stupid()
nerd_1.instance_variables() # => [:@name, :@age, :@stupid]
```

## 深入Ruby类

在Ruby中，类具有双重身份：

1. 与字面意思一致，代表对象的类型。
1. 类本身也是一个对象 – Class 类型的一个实例。

类的第二重身份，使得ruby能够以很优雅的方式实现类实例变量(class instance variable)和类方法(class method)。

```ruby
class Nerd
    @nerd_count = 0

    def self.increase_count
       @nerd_count += 1 
    end
end

Nerd.class # => Class
Nerd.instance_variables() # => [:@nerd_count]
Nerd.singleton_methods() # => [:increase_count]
```

相应的对象模型为：

![](/images/ruby_class_model.jpg)

后面学习到单件类(singleton class)的时候， 我们可以深入理解class的具体模型，目前只需要知道class具有双重身份即可。