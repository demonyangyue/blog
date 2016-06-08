---
layout: post
title: Ruby元编程系列2 – 方法
date: 2015-04-22
categories: Programming
tags: [Ruby, Meta Programming]
---

## 引言

在一次简单地方法调用背后，ruby需要做大量的工作，让我一起开启探索ruby方法内幕的旅程吧。

和其它面向对象的语言一样，ruby在调用一个方法也要经过两个步骤：

1. 找到这个方法 —— 方法查找
1. 执行这个方法——在当前对象(self)上调用这个方法

<!--more-->

## 当前对象——self

当前对象(self)是理解ruby方法调用的核心概念。每一次ruby方法调用都需要在某个对象上执行，这个对象被称为方法的接受者 (receiver), 也叫作当前对象(self).

在任何时刻，只有一个对象能充当当前对象，但没有哪个对象能长期充当这一角色。要想成为优秀的ruby程序员，你需要知道任何时刻当前哪个对象在充当self的角色。

```ruby
#!/usr/bin/ruby

class Nerd

   puts self # => Nerd 

   def print_self
      puts self # => #<Nerd:0x00000000e3f8a0>
   end

end

nerd_1 = Nerd.new
puts nerd_1.print_self()
```

从上面代码实例可以看出，在定义类的时候，self为当前类(Nerd), 在方法调用的过程中，self为当前对象(Nerd 类的实例).


## 方法查找

Ruby的方法查找可以用一句话概括： 首先在接收者的类中查找，然后一层层地沿着继承链向上查找。

```ruby
#!/usr/bin/ruby

#!/usr/bin/ruby

class Nerd
    def parent_method
        puts "in parent"
    end
  end

class SubNerd < Nerd
    def child_method
        puts "in child"
    end
end

sub_nerd_1 = SubNerd.new
sub_nerd_1.parent_method() # => "in parent"
sub_nerd_1.child_method() # => "in child"
```

方法查找的过程如图所示：

![](/images/ruby_method_lookup.jpg)

这种查找方法也被简单概括为”向右一步，再向上”规则。

除了画图以外，也可以通过自省——调用ancestors()方法来获得一个类的继承链。

 ```ruby
  SubNerd.ancestors() # => [SubNerd, Nerd, Object, Kernel, BasicObject]
 ```

## 动态方法定义

在运行过程中动态创建组件是元编程的特点。可以通过动态方法定义，来消除定义相似方法时产生的重复代码。

假设我们在Nerd类中定义了两个简单的方法：

```ruby
#!/usr/bin/ruby

class Nerd
    attr_reader :name
    def initialize
       @name = "yue" 
       @age = 25
    end
    
    def print_name
        puts "The name is #{@name}"
    end

    def print_age
        puts "The name is #{@name}"
    end
end
```

可以利用Module#define_mthod()来定义一个方法，只需提供一个方法名和充当方法主体的块：

```ruby
#!/usr/bin/ruby

class Nerd
    attr_reader :name
    def initialize
       @name = "yue" 
       @age = 25
    end

    
    ["name", "age"].each do |attr|
        define_method("print_#{attr}") do 
            variable = "@#{attr}" 
            puts "The name is #{instance_variable_get(variable)}"
        end
    end   
end

nerd_1 = Nerd.new
nerd_1.print_name
nerd_1.print_age
```

## 动态方法调用

通常的方法调用通过(.)操作符，将方法发送给指定的对象。

ruby 提供了Object#send()方法，来支持动态方法调用。通过send()方法，可以将需要调用的方法名当作参数传递，增加了一层间接性，使方法调用变得更加灵活。这种技术被称为动态派发(Dynamic Dispatch)。

在上一节的代码示例中，可以使用动态方法调用消除最后两行代码的重复性：

```ruby
nerd_1 = Nerd.new
["name", "age"].each do |attr|
    nerd_1.send("print_#{attr}")    
end
```

## 幽灵方法

当需要定义许多类似的方法时，可以通过只定义method_missing()方法来消除重复。如果方法查找没有找到对应的方法，将会触发method_missing()方法。

从调用者的角度看，这和调用普通方法没有区别，但实际上接受者并没有明确定义相对应的方法。因此method_missing()也被称为幽灵方法(Ghost Method).

下面是一个用幽灵方法来模拟动态代理的示例：

```ruby
#!/usr/bin/ruby

class NerdProxy
    def sleep
       puts "sleeping" 
    end    

    def play
       puts "playing" 
    end
end

class Nerd
    attr_reader :name

    def initialize
       @name = "yue" 
       @age = 25
       @proxy = NerdProxy.new
    end
    
    def code
           puts "coding"
    end
   
    def method_missing(name, *args)
        @proxy.send(name) 
    end
end

nerd_1 = Nerd.new
nerd_1.code
nerd_1.sleep
```

通过method_missing 方法，将Nerd类未定义的方法转发给代理类。

关于ruby方法我们就研究这么多，下一篇将研究ruby的作用域。