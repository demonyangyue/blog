---
layout: post
title: Rails源码管窥5 - 添加属性方法
---

## 引言
这一节我们研究rails MVC 架构中的model部分，rails 中有三种model相关的类型：`ActiveModel, ActiveRecord 和 ActiveResource`。

`ActiveModel`实现了和model相关的许多功能，比如`validation`,`ActiveRecord`在`ActiveModel`的基础上，提供了和数据库相关的ORM支持，`ActiveResource`在`ActiveModel`的基础上，提供了对`WebService API`的支持。本节我们关注`ActiveModel`相关的模块。

## 添加Attribute Methods

`ActiveModel::AttributeMethods`可以给类动态添加属性方法。
你经常会遇到这样的情况：一个类具有一些属性，你需要为这些属性分别创建名字和功能类似的方法，这些方法名具有相同的前缀或后缀，这时候就可以通过包含`ActiveModel::AttributeMethods`类型来消除重复。

来看一个实际的例子：

```ruby
  class Person
    include ActiveModel::AttributeMethods

    attribute_method_suffix '_contrived?'
    attribute_method_prefix 'clear_'
    define_attribute_methods :name

    attr_accessor :name

    def attributes
      { 'name' => @name }
    end

    private

    def attribute_contrived?(attr)
      true
    end

    def clear_attribute(attr)
      send("#{attr}=", nil)
    end

  end
```

没有任何显式的定义，我们自动拥有了`name_contrived`,`clear_name`等方法，`ActiveModel::AttributeMethods`是如何实现这样的魔法的呢？让我们到源代码中一探究竟吧。

先来看看最重要的`define_attribute_methods()`方法：

```ruby

      def define_attribute_methods(*attr_names)
=>        attr_names.flatten.each { |attr_name| define_attribute_method(attr_name) }
      end

```

为每个`attribute`调用`define_attribute_method()`方法：

```ruby

      def define_attribute_method(attr_name)
        attribute_method_matchers.each do |matcher|
          method_name = matcher.method_name(attr_name)

        unless instance_method_already_implemented?(method_name)
            generate_method = "define_method_#{matcher.method_missing_target}"

            if respond_to?(generate_method, true)
              send(generate_method, attr_name)
            else
=>            define_proxy_call true, generated_attribute_methods, method_name, matcher.method_missing_target, attr_name.to_s
            end
          end
        end
        attribute_method_matchers_cache.clear
      end
```

对于我们上面的例子来说，由于并没有在类中显式定义`clear_name()`方法，所以`instance_method_already_implemented?(method_name)`返回`false`，
并且由于`define_method_clear_attribute()`方法也没有定义，实际执行的是`define_proxy_call`这一行：

```ruby

        def define_proxy_call(include_private, mod, name, send, *extra) #:nodoc:
          defn = if name =~ NAME_COMPILABLE_REGEXP
            "def #{name}(*args)"
          else
            "define_method(:'#{name}') do |*args|"
          end

          extra = (extra.map!(&:inspect) << "*args").join(", ")

          target = if send =~ CALL_COMPILABLE_REGEXP
            "#{"self." unless include_private}#{send}(#{extra})"
          else
            "send(:'#{send}', #{extra})"
          end

          mod.module_eval <<-RUBY, __FILE__, __LINE__ + 1
            #{defn}
              #{target}
            end
          RUBY
        end

```

这里使用代码字符串来实现方法的动态定义，结合我们的例子，这段代码最终会被翻译成：

```ruby

define_method(:clear_name) do |*args|
  send(:clear_attribute, 'name')
end

```

至此我们已经理解了`ActiveModel::AttributeMethods`动态定义属性方法的内幕。

神奇的是，当我们在Person类的定义中，注释掉`define_attribute_methods :name`这一行，依然可以获得`clear_name()`的定义，这是如何做到的呢？

我们在源代码中找到了`method_missing()`这个hook：

```ruby
    def method_missing(method, *args, &block)
      if respond_to_without_attributes?(method, true)
        super
      else
        match = match_attribute_method?(method.to_s)
        match ? attribute_missing(match, *args, &block) : super
      end
    end

    def attribute_missing(match, *args, &block)
      __send__(match.target, match.attr_name, *args, &block)
    end
```

通过`method_missing()`,将对`#{prefix}#{attr}#{suffix}(*args, &block)`的调用转换成了对`#{prefix}attribute#{suffix}(#{attr}, *args, &block)`的调用，从而找到了对应的方法定义。

动态添加属性方法就暂且研究到这里，下一节我们将研究ActiveModel中validation实现的机制。


