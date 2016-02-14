---
layout: post
title: Rails源码管窥3 - 模板查找
date: 2015-06-1
categories: Programming
tags: [Rails, Web]
---

## 引言
上一篇中通过对render函数调用栈的学习，我们理解了controller在调用render函数时，如何找到renderer并渲染到相应的template。但是rails是如何知道应该渲染哪个模板的呢？这正是本节我们关注的问题。

<!--more-->
##  Rails模板查找机制
在上一节，我们提到`_render_template()`方法的定义为：
>  /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/rendering.rb

```ruby   
   94:       def _render_template(options) #:nodoc:
   95:         variant = options[:variant]
   96: 
   97:         lookup_context.rendered_format = nil if options[:formats]
   98:         lookup_context.variants = variant if variant
   99: 
=> 100:        view_renderer.render(view_context, options)
   101:      end
   102: 
```

在执行`view_renderer.render(view_context,optoins)`时，通过@lookup_cotext的`find()`方法，来查找相应的template：

>  /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/lookup_context.rb

```ruby
    120:       def find(name, prefixes = [], partial = false, keys = [], options = {}    )
=>  121:         @view_paths.find(*args_for_lookup(name, prefixes, partial, keys, options))
    122:       end
    123:       alias :find_template :find
```

进而调用定义在PathSet类中的`find()`和`find_all()`方法：

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/path_set.rb

```ruby
    45     def find(*args)
    46       find_all(*args).first || raise(MissingTemplate.new(self, *args))
    47     end
    48
    49     def find_all(path, prefixes = [], *args)
    50       prefixes = [prefixes] if String === prefixes
    51       prefixes.each do |prefix|
    52         paths.each do |resolver|
=>  53           templates = resolver.find_all(path, prefix, *args)
    54           return templates unless templates.empty?
    55         end
    56       end
    57       []
    58     end
```

最终我们在ActionView::Resolver类中找到`find_all()`方法的定义：

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/template/resolver.rb

```ruby
    114    def find_all(name, prefix=nil, partial=false, details={}, key=nil, locals=[])
    115     cached(key, [name, prefix, partial], details, locals) do
=>  116        find_templates(name, prefix, partial, details)
    117      end
    118    end
```

`cached()`方法查找template的缓存，如果找到则直接返回template，否则执行`find_templates()`方法。关于template的缓存机制，我们在下一节详细介绍。

`find_templates()`方法并没有在ActionView::Resolver中实现，而是在其子类中定义。如果我们想要定制自己的resolver,比如希望从数据库而非文件系统中查找template，那么就需要在自定义的resolver类型中定义`find_templates()`方法，根据传入`name, prefix, partial, details`参数，从数据库中取出对应的条目，生成一个`ActionView::Template`对象并返回.

Rails中默认的resolver是ActionView::FileSystemResolver，从文件系统中查找template，其`find_templates()`方法定义为：

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/template/action_view/resolver.rb

```ruby
=>  177     def find_templates(name, prefix, partial, details)
    178       path = Path.build(name, prefix, partial)
    179       query(path, details, details[:formats])
    180     end
    181
    182     def query(path, details, formats)
    183       query = build_query(path, details)
    184
    185       template_paths = find_template_paths query
    186
    187       template_paths.map { |template|
    188         handler, format, variant = extract_handler_and_format_and_variant(template, formats)
    189         contents = File.binread(template)
    190
    191         Template.new(contents, File.expand_path(template), handler,
    192           :virtual_path => path.virtual,
    193           :format       => format,
    194           :variant      => variant,
    195           :updated_at   => mtime(template)
    196         )
    197       }
    198     end
```

当浏览器中访问的url为`users/new`时，rails默认生成`FileSystemResolver`类型的一个实例：` FileSystemResolver.new("/path/to/views", ":prefix/:action{.:locale,}{.:formats,}{+:variants,}{.:handlers,}")`, 最终实际的template查找路径为`app/views/users/new{.{en},}{.{html,js},}{.{erb,haml},}`。

我们来详细研究一下`FileSystemResolver`的参数中每个字段的含义：

```ruby
  :prefix - 通常为controller的路径
  :action - action 的名字
  :locale - 可能的locale（例如 en）
  :formats - 可能的请求类型 (例如 html, json, xml...)
  :variants - 可能的请求变量 (例如 phone, tablet...)
  :handlers - 可能的模板处理器 (例如 erb, haml, builder...)
```

默认的模板查找起始路径`/path/to/views`为`app/views`, 可以通过`ActionView::ViewPaths`中提供的`append_view_path()`方法添加自定义的起始查找路径。

现在我们已经理解了Rails的模板查找机制，下一节我们将深入研究Rails的模板缓存机制。
