---
layout: post
title: Rails源码管窥6 - 数据验证
date: 2015-06-17
categories: Programming
tags: [Rails, Web]
---

## 引言
在web开发的过程中，经常需要对数据的有效性进行验证，以保证数据符合我们的需求。

数据验证可以在以下几处完成：

1. 在前端通过JavaScript进行验证。Javascript最初被创造的目的就是为了对表单进行数据验证，使用JS验证的优点是简单，响应迅速，缺点在于无法对涉及后端逻辑的数据进行验证。

1. 在Model层进行验证。当数据验证需要在后端完成时，这是推荐的做法。

1. 在Controller层进行验证。这种做法的主要缺点在于会使controller的代码膨胀，并且增加与model层的耦合度，不利于代码的阅读和维护。

1. 在数据库层面进行验证。数据库也提供了constraints功能来对保证数据的正确性，但是缺点主要在于提供的验证方式有限，并且直到数据最终存储到数据库前，才能发现错误。

Rails通过ActiveModel模块提供了一系列预定义的类来辅助验证，用户也可以根据自己的需求来自定义验证逻辑。

<!--more-->
## ActiveModel 数据验证

看一个简单的例子，假设我们需要定义一个`Person`类型，要求其实例必须具有`:name`字段：

```ruby

class Person
  include ActiveModel::Validations

  validates :name, presence: true

  attr_accessor :name
end

```

于是我们可以创建Person类型的实例并验证其有效性：

```bash

2.0.0-p353 :002 > person = Person.new
 => #<Person:0xb3c2a98> 
2.0.0-p353 :003 > person.name = 'yy'
 => "yy" 
2.0.0-p353 :004 > person.valid?
 => true 

```

那么`validates :name, presence: true`在背后是怎样定义验证逻辑的呢？让我们来一探究竟吧：

当调用`validates()`方法时，通过传入的参数解析出对应的validator，并进一步调用`validates_with()`方法：

> /usr/local/rvm/gems/ruby-2.2.1/gems/activemodel-4.2.1/lib/active_model/validations/validates.rb

```ruby
      
      def validates(*attributes)
        defaults = attributes.extract_options!.dup
        validations = defaults.slice!(*_validates_default_keys)

        raise ArgumentError, "You need to supply at least one attribute" if attributes.empty?
        raise ArgumentError, "You need to supply at least one validation" if validations.empty?

        defaults[:attributes] = attributes

        validations.each do |key, options|
          next unless options
          key = "#{key.to_s.camelize}Validator"

          begin
            validator = key.include?('::') ? key.constantize : const_get(key)
          rescue NameError
            raise ArgumentError, "Unknown validator: '#{key}'"
          end

=>        validates_with(validator, defaults.merge(_parse_validates_options(options)))
        end
      
```

`validates_with()`方法创建相应的`validator`实例并调用`validate()`方法：

> /usr/local/rvm/gems/ruby-2.2.1/gems/activemodel-4.2.1/lib/active_model/validations/with.rb

```ruby

      def validates_with(*args, &block)
        options = args.extract_options!
        options[:class] = self

        args.each do |klass|
          validator = klass.new(options, &block)

          if validator.respond_to?(:attributes) && !validator.attributes.empty?
            validator.attributes.each do |attribute|
              _validators[attribute.to_sym] << validator
            end
          else
            _validators[nil] << validator
          end

=>        validate(validator, options)
        end
      end
```

`validate()`对传入的参数进行检验，并注册`Validator::validate()`作为回调函数：

> /usr/local/rvm/gems/ruby-2.2.1/gems/activemodel-4.2.1/lib/active_model/validations.rb

```ruby

      def validate(*args, &block)
        options = args.extract_options!

        if args.all? { |arg| arg.is_a?(Symbol) }
          options.each_key do |k|
            unless VALID_OPTIONS_FOR_VALIDATE.include?(k)
              raise ArgumentError.new("Unknown key: #{k.inspect}. Valid keys are: #{VALID_OPTIONS_FOR_VALIDATE.map(&:inspect).join(', ')}. Perhaps you meant to call `validates` instead of `validate`?")
            end
          end
        end

        if options.key?(:on)
          options = options.dup
          options[:if] = Array(options[:if])
          options[:if].unshift ->(o) {
            Array(options[:on]).include?(o.validation_context)
          }
        end

        args << options
=>      set_callback(:validate, *args, &block)
```

当调用类似`person.valid?`这样的方法时，`Validator：：validate()`函数会被触发, 调用`validate_each()`方法：

> /usr/local/rvm/gems/ruby-2.2.1/gems/activemodel-4.2.1/lib/active_model/validator.rb

```ruby

        # Performs validation on the supplied record. By default this will call
        # +validates_each+ to determine validity therefore subclasses should
        # override +validates_each+ with validation logic.
        def validate(record)
          attributes.each do |attribute|
            value = record.read_attribute_for_validation(attribute)
            next if (value.nil? && options[:allow_nil]) || (value.blank? && options[:allow_blank])
=>          validate_each(record, attribute, value)
          end

```

每个validator都实现了自己的`validate_each()`方法，接受`record, attr_name, value`三个参数，执行数据校验：

> /usr/local/rvm/gems/ruby-2.2.1/gems/activemodel-4.2.1/lib/active_model/validations/presence.rb

```ruby

    2: module ActiveModel
    3: 
    4:   module Validations
    5:     class PresenceValidator < EachValidator # :nodoc:
    6:       def validate_each(record, attr_name, value)
=>  7:         record.errors.add(attr_name, :blank, options) if value.blank?
    8:       end
    9:     end
```

## 自定义验证类：

`ActiveModel::Validations`模块提供了多种预定义的`Validator`,例如`PresenceValidator`,`InclusionValidator`,`NumericalityValidator`，如果我们想自定义`Validator`,只需依葫芦画瓢，继承`EachValidator`并且实现对应`validate_each()`方法即可：

```ruby
     class TitleValidator < ActiveModel::EachValidator
       def validate_each(record, attribute, value)
         record.errors.add attribute, 'must be Mr., Mrs., or Dr.' unless %w(Mr. Mrs. Dr.).include?(value)
       end
     end
```

关于数据验证我们就研究到这里，下一节我们将探索Rails中的autoload机制。
