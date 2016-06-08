---
layout: post
title: Ruby元编程系列4 – 单件方法和模块
date: 2015-04-24
categories: Programming
tags: [Ruby, Meta Programming]
---

## 单件方法

单件方法可能是Ruby元编程中最重要的一个概念，通过单件方法，可以给单个对象增加方法。

<!--more-->

首先需要指出的是，这里的“单件”和设计模式中的“单件模式”没有任何关系，不要混淆它们。

假设现在需要给一个字符串对象添加一个greet 方法，我们可以有两种解决方案：

1. 打开String 类，添加greet方法
2. 给字符串对象添加greet单件方法

第一种方法虽然可以达到我们的目的，但是会污染整个String类，是得其它的String类型的对象也拥有了greet方法。

我们来看看更优雅的第二种方案：

```ruby
#!/usr/bin/ruby

name = "yue"

def name.greet
   "hi " + self 
end

puts name.greet # => "hi yue"
```

是不是很酷？
聪明的你可能会问，在对象模型内部，单件方法是放在哪里的呢？

一定不会是在对象所属的类中，因为那样的话该方法会被所有同类型的对象所拥有，不再专属于这个对象了。
为了回答这个问题，让我们一起揭开单件方法的神秘面纱吧。

其实，在类型继承链的底部，Ruby偷偷的藏了一个单件类（也称作eigenclass）， 单件方法被放在单件类中。

单件类是没有名字的，我们暂且用#name来标记它，如果你想知道ruby在私下是如何称呼单件类的，可以通过一种特殊的基于class关键字的语法，来进入到单件类的内部：

```ruby
name = "yue"

singleton_class = class << name
    self
end
puts singleton_class # => #<Class:#<String:0x00000002813ab0>> 
```

很绕口的名字对不对？

我们现在可以画出实际的内部对象模型：

![](/images/singleton_method.jpg)

## 类方法

之前我们已经讲过，类有两重身份，第二种身份是说类本身也是一个对象，是Class类型的一个实例，那么在这个对象上定义单件方法，我们就能够获得专属于这个类的类方法。

来看一个最简单的示例：

```ruby
#!/usr/bin/ruby

class Nerd
    def self.greet
        puts "hello world"
    end
end

Nerd.greet
```

对象模型：

![](/images/class_method.jpg)

## 类宏

对于类方法，Ruby提供了一种语法糖，形式类似于C语言中的宏，我们把它称之为类宏。

Ruby对于属性的方法控制，提供了两个拟态方法：attr_reader 和attr_accessor, 这是类宏的典型应用，我们在之前的代码中已经看到了它们的使用方式：

```ruby
class Nerd
    attr_reader :name
    attr_accessor :age
end
```

它们是怎么实现的呢？我们可以自己用代码来模拟一下：

```ruby
#!/usr/bin/ruby

class Nerd

    def self.my_attr_accessor(name)
       
        define_method("#{name}") do 
           puts "define getter: #{name}" 
           instance_variable_get("@#{name}")
        end

        define_method("#{name}=") do |val| 
           puts "define setter: #{name}=" 
           instance_variable_set("@#{name}", val)
        end
    end

    my_attr_accessor :name

    def initialize()
       @name = "yang" 
       @age = 25
    end

end

```
看到了吧， Ruby元编程真是太酷了！

## 模块

Ruby 模块（Module）本质上是类的特例，相比于类，模块没有定义new方法，所以不能被实例化，没有定义superclass（）方法，所以不能被继承：

```ruby
Class.new.methods - Module.new.methods #=> [:allocate, :new, :superclass]
```

当在类定义中include一个模块时，模块的方法会成为类的实例方法，当在类中extend一个模块时，模块的方法会成为类的类方法。

那么如果我在模块中定义了一些方法，希望其中一部分成为类的实例方法，另外一部分成为类的方法，应该怎么做呢？需要定义两个模块么？

强大的Ruby元编程提供了一种称之为类扩展混入的技术，代码示例;

```ruby
#!/usr/bin/ruby

module MyMixin
    module ClassMethods
        def class_method
            puts "this in a class method" 
        end
    end

    module InstanceMethods
        def instance_method
            puts "this is a instance method" 
        end

    end

    def self.included(receiver)
        receiver.extend         ClassMethods
        receiver.send :include, InstanceMethods
    end
end

```

当某个类include该模块时，self.included这个hook方法被触发，ClassMethods中定义的方法会作为类方法被加入，InstanceMethods中定义的方法会作为实例方法被加入。

Ruby元编程系列就先讲这么多，后续会讲元编程在Rails代码中的应用，敬请期待！