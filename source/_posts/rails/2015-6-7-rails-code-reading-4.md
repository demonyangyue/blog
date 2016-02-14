---
layout: post
title: Rails源码管窥4 - 模板缓存
date: 2015-06-07
categories: Programming
tags: [Rails, Web]
---

## 引言
通过上一节的研究，我们理解了Rails模板查找机制。
但复杂的查找逻辑也意味着昂贵的时间开销，那么如何提高模板查找的效率呢？聪明的你一定已经想到了——缓存查找结果。

我们该如何实现rails的模板缓存系统呢？
最简单的办法是使用一个hash ,但是查找结果由`name, prefix, partial, details`这些参数共同决定， 那么hash key该如设计呢？

让我们一起到Rails源码中寻找答案吧。

<!--more-->
## Rails 模板缓存机制

上一节中我们提到了ActionView::Resolver类中`find_all()`方法的定义：

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/template/resolver.rb

```ruby
    def find_all(name, prefix=nil, partial=false, details={}, key=nil, locals=[])
      cached(key, [name, prefix, partial], details, locals) do
        find_templates(name, prefix, partial, details)
      end
    end
```

其中`cached()`方法查找template的缓存，如果找到则直接返回template，否则执行`find_templates()`方法。

我们首先想知道，传进来的`key`参数是如何计算的，我们可以在`ActionView::LookupContext::DetailsKey`中找到相关定义，代码等价于：

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/lookup_context.rb

```ruby
        key = @details_key[details] || = Object.new
```

可以看到，`DetailsKey`中使用一个简单的`Object`实例，作为`key`变量的值。为什么不直接用`details`作为`key`变量的值呢？其中的奥秘在于 —— **性能**。

```ruby
#!/usr/bin/ruby

require 'benchmark'

key_1 = Object.new
key_2 = {
    formats: [:html],
    locale: [:en, :en],
    handler: [:erb, :builder, :rjs]
}

hash_1 = {key_1 => true}
hash_2 = {key_2 => true}

Benchmark.realtime{ 1000.times {hash_1[key_1]}} # => 0.001037
Benchmark.realtime{ 1000.times {hash_2[key_2]}} # => 0.011679
```

使用简单的`Object`实例作为`key`变量的值，性能提高了10倍。

下面来看`cached()`方法的定义：

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/template/resolver.rb

```ruby

    # Handles templates caching. If a key is given and caching is on
    # always check the cache before hitting the resolver. Otherwise,
    # it always hits the resolver but if the key is present, check if the
    # resolver is fresher before returning it.
    def cached(key, path_info, details, locals) #:nodoc:
      name, prefix, partial = path_info
      locals = locals.map { |x| x.to_s }.sort!

      if key
        @cache.cache(key, name, prefix, partial, locals) do
          decorate(yield, path_info, details, locals)
        end
      else
        decorate(yield, path_info, details, locals)
      end
    end
```

其中`decorate()`方法，为先前`find_templates`方法中返回的templates设置相应的属性。我们重点关注`@cache.cache()`方法：

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/template/resolver.rb

```ruby

      # Cache the templates returned by the block
      def cache(key, name, prefix, partial, locals)
        if Resolver.caching?
          @data[key][name][prefix][partial][locals] ||= canonical_no_templates(yield)
        else
          fresh_templates  = yield
          cached_templates = @data[key][name][prefix][partial][locals]

          if templates_have_changed?(cached_templates, fresh_templates)
            @data[key][name][prefix][partial][locals] = canonical_no_templates(fresh_templates)
          else
            cached_templates || NO_TEMPLATES
          end
        end
      end
```

如果`Resolver.caching?`为true，则直接将templates缓存，否则，将找到templates和已缓存的作对比，如有变化则更新缓存。

这里有趣的地方是，我们发现最终缓存template使用的key是`[key][name][prefix][partial][locals]`,为什么不直接用数组`[key, name, prefix, partial, locals]`呢？ 奥秘依然在于性能：

```ruby
#!/usr/bin/ruby

require 'benchmark'

key= Object.new
name = "index"
prefix = ["home", "application"]
partial = ""
locals = [:en, :en]

hash_1  = Hash.new 
hash_2 = { [key, name, prefix, partial, locals] => true }

puts Benchmark.realtime{ 1000.times do 
    hash_1.fetch('key', {}).fetch('name', {}).fetch('prefix', {}).fetch('partial', {}).fetch('locals', true)
end
} # => 0.00256
puts Benchmark.realtime{ 1000.times do 
    hash_2[[key, name, prefix, partial, locals]]
end
} # => 0.01112
```

可以看到，用多个简单的对象作为键，比用一个大的数组作为键要高效得多。

## 结束语

关于Rails模板缓存，一个值得探索的领域是在分布式系统中，如何对缓存进行管理，未来会结合memcached之类的分布式内存管理框架进一步研究。让我们暂且放下模板缓存，开始下一段探索之旅吧。
