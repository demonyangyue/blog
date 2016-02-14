---
layout: post
title: Rails源码管窥2 - render内幕
date: 2015-05-28
categories: Programming
tags: [Rails, Web]
---

## 引言
Model-View-Controller(MVC)是Rails的核心架构。在Controller定义的方法中，最后一步通常是调用render函数，将Controller生成的数据渲染到指定的template， 例如：`format.json { render json: @article.errors, status: :unprocessable_entity }` , 那么render函数在调用时，幕后做了什么呢？让我们一起开启探索之旅吧。

## Render函数调用栈
开启Byebug进入调试模式，输入step（s）进入render方法内部：
> /usr/local/rvm/gems/ruby-2.2.1/gems/actionpack-4.2.1/lib/action_controller/metal/instrumentation.rb

```ruby
   41:     def render(*args)
   42:       render_output = nil
   43:       self.view_runtime = cleanup_view_runtime do
=> 44:         Benchmark.ms { render_output = super }
   45:       end
   46:       render_output
   47:     end
   48: 

```
发现render函数嵌在Benchmark.ms方法中，于是我们就可以知道每一次render消耗的时间咯。

从44行可以看出，实际在调用父类的render（）方法， step进去一探究竟吧：

>  /usr/local/rvm/gems/ruby-2.2.1/gems/actionpack-4.2.1/lib/abstract_controller/rendering.rb

```ruby
   20:     # Normalize arguments, options and then delegates render_to_body and
   21:     # sticks the result in self.response_body.
   22:     # :api: public
   23:     def render(*args, &block)
=> 24:       options = _normalize_render(*args, &block)
   25:       self.response_body = render_to_body(options)
   26:       _process_format(rendered_format, options) if rendered_format
   27:       self.response_body
   28:     end
```

`_normalize_render()`函数的主要作用是将用户传入的参数转换成hash。
那么`render_to_body()`做了什么呢？

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionpack-4.2.1/lib/action_controller/metal/renderers.rb

```ruby
   36:     def render_to_body(options)
=> 37:       _render_to_body_with_renderer(options) || super
   38:     end
   39: 
```

如果在`/metal/renderers.rb`中定义了相应格式（如json,xml）的render，则`render_to_body()`会调用定义的诸如`_render_with_renderer_json`方法。

> /usr/local/rvm/gems/ruby-2.2.1/gems/actionpack-4.2.1/lib/action_controller/metal/renderers.rb

```ruby
   40:     def _render_to_body_with_renderer(options)
   41:       _renderers.each do |name|
   42:         if options.key?(name)
   43:           _process_options(options)
=> 44:           method_name = Renderers._render_with_renderer_method_name(name)
   45:           return send(method_name, options.delete(name), options)
```

如果没有定义相应格式的render， 则`render_to_body()`会沿着继承链一层层向上转发，最后到达`ActionView::Rendering`类：
>  /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/rendering.rb

```ruby
   81:     def render_to_body(options = {})
   82:       _process_options(options)
=> 83:       _render_template(options)
   84:     end
```

`_render_template()`方法的定义为：
>  /usr/local/rvm/gems/ruby-2.2.1/gems/actionview-4.2.1/lib/action_view/rendering.rb

```ruby   
   94:       def _render_template(options) #:nodoc:
=> 95:         variant = options[:variant]
   96: 
   97:         lookup_context.rendered_format = nil if options[:formats]
   98:         lookup_context.variants = variant if variant
   99: 
   100:        view_renderer.render(view_context, options)
   101:      end
   102: 
```

最终的模板渲染由`view_renderer.render(view_context,options)`完成，其中view_context是模板解析的上下文，是`ActionView::Base`类型的实例。

到此为止，我们对render函数已经有了比较完整的认识，就在这里停下吧~

## 挑战 —— 卸载多余模块

在rails中，由Module组成的继承链是独具匠心的设计，可以让我们按照需求方便地添加和删除功能。

比如在render函数调用时，我想移除Benchmark相关的功能，可以怎么做呢？

Rails 每一个子模块，都有一个基本的base类型，比如ActionController的基本类型为ActionController::Base, 定义在`gems/abstract_controller/base.rb`中。在定义Base时，通过include模块的方式，定义了继承链，动态添加所需的功能。

```ruby
  class Base < Metal
    abstract!

    def self.without_modules(*modules)
      modules = modules.map do |m|
        m.is_a?(Symbol) ? ActionController.const_get(m) : m
      end

      MODULES - modules
    end

    MODULES = [
      AbstractController::Rendering,
      AbstractController::Translation,
      AbstractController::AssetPaths,

      Helpers,
      HideActions,
      UrlFor,
      Redirecting,
      ActionView::Layouts,
      Rendering,
      Renderers::All,
      ConditionalGet,
      EtagWithTemplateDigest,
      RackDelegation,
      Caching,
      MimeResponds,
      ImplicitRender,
      StrongParameters,

      Cookies,
      Flash,
      RequestForgeryProtection,
      ForceSSL,
      Streaming,
      DataStreaming,
      HttpAuthentication::Basic::ControllerMethods,
      HttpAuthentication::Digest::ControllerMethods,
      HttpAuthentication::Token::ControllerMethods,

      # Before callbacks should also be executed the earliest as possible, so
      # also include them at the bottom.
      AbstractController::Callbacks,

      # Append rescue at the bottom to wrap as much as possible.
      Rescue,

      # Add instrumentations hooks at the bottom, to ensure they instrument
      # all the methods properly.
      Instrumentation,

      # Params wrapper should come before instrumentation so they are
      # properly showed in logs
      ParamsWrapper
    ]

    MODULES.each do |mod|
      include mod
    end
```

如果我们需要移除BenchMark模块，可以这样定义controller:

```ruby
    class MyController
        ActionController::Base.without_modules(:BenchMark).each do |left|
            include left
        end
    end
```
调试MyController时，会发现不再有性能分析相关的代码被执行。

关于render函数的调用，我们暂时就研究到这里，下一节让我们一起探索rails中的模板查找机制吧~
