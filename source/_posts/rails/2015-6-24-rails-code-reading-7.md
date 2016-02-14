---
layout: post
title: Rails源码管窥7 - autoload 机制
date: 2015-06-24
categories: Programming
tags: [Rails, Web]
---

## 引言
在开发大型web应用时，一个普遍的痛点是在每个源代码文件的开头，都需要一长串的import来引入所需的类，使得源代码变得冗长并且容易出错。
Rails通过`ActiveSupport::Autoload`模块，免去了手动加载类或者模块的烦恼。Rails的autoload机制主要提供给了两大功能：

* 自动加载类和模块的定义

 当rails看到未定义的常量时，会根据名字猜测其所处的路径，然后加载其定义。这里又一次体现了rails `convention over configuration`哲学的强大力量。

 Rails采用缓式(lazy)加载的策略，在常量名首次出现时试图尝试其定义。缓式加载使得rails server在开发模式启动时，只需加载少量的类及模块定义，整个启动过程可在较短时间内完成。


* 文件修改后自动重新加载

 当处于开发模式时，如果开发者修改了controller, model等目录下的源码文件，无需重启服务器，下一次请求rails会自动读取最新的定义，后面我们会一起探索这样的魔法是如何实现的。

<!--more-->
## ruby中的autoload

ruby本身通过`Kernel#autoload`方法，提供了常量自动加载机制，例如：

```ruby
autoload :Command,            'thin/command'
autoload :Connection,         'thin/connection'
```
当ruby第一次看到`Command`时，由于不知其定义，`Module#const_missing`方法被触发，进而根据autoload指定的路径`thin/command`加载定义。

## rails中的autoload

Rails扩展了ruby的autoload功能，用户可以不显式指定路径，rails通过会自动猜测路径，例如：

```ruby
module ActiveSupport
  extend ActiveSupport::Autoload

  autoload :Concern
  autoload :Dependencies
end
```

那么rails是如何猜测路径的呢？原来，rails有一个`autoload_paths`的配置，来指定常量定义的搜索路径。启动一个rails console：

```bash
2.2.1 :001 > ActiveSupport::Dependencies.autoload_paths
 => [".../app/assets", 
 ".../app/controllers",
 ".../app/helpers", 
 ".../app/mailers", 
 ".../app/models", 
 ".../app/controllers/concerns", 
 ".../app/models/concerns",
 ".../test/mailers/previews"] 
```
可以看到autoload_paths所指定的一些列搜索路径。

让我们深入到autoload的源代码中了解其定义：

> /usr/local/rvm/gems/ruby-2.2.1/gems/activesupport-4.2.1/lib/active_support/dependencies/autoload.rb

```ruby
    def autoload(const_name, path = @_at_path)
      unless path
        full = [name, @_under_path, const_name.to_s].compact.join("::")
        path = Inflector.underscore(full)
      end

      if @_eager_autoload
        @_autoloads[const_name] = path
      end

      super const_name, path
    end

```

可以看到，rails中通过`path = Inflector.underscore(full)`来自动推断constant所处的路径，例如在查找`ApplicationController`的定义时，
首先检查`app/assets/application_controller.rb`是否存在，发现不存在后继续查找`app/controllers/application_controller.rb`文件，得到路径后，
通过`super const_name, path`来调用`Kernel#autoload`.

## Eager Load

Rails通过缓式加载，让常量在第一次使用时加载定义，可以在开发环境中加快Server的启动速度。但是在生产环境中，采用缓式加载的策略就会使处理用户请求的时间变长，造成性能下降。
rails在生产环境中采用即时加载（eager load）策略，在server启动时预先加载大部分类和模块的定义。

Rails通过变量`config.cache_classes`来切换常量加载策略，当`config.cache_classes`为true时，调用`Kernel#require`来加载，这是生产模式默认的配置，当`config.cache_classes`为false时，调用`Kernel#load`来加载，这是开发模式默认的配置。`Kernel#load`允许rails多次加载同一文件。

当运行在开发模式时，如果想eager load某个模块，该如何做呢？别担心，`ActiveSupport::Autoload`为我们提供了`eager_autoload`方法：

```ruby
module ActiveSupport
  extend ActiveSupport::Autoload

  eager_autoload do
    autoload :BacktraceCleaner
  end
end
```

## Reload

当在开发环境中运行rails程序时，如果相关的代码发生了改动，rails会自动重新加载代码的定义。Rails会监控以下文件的变动：

* config/routes.rb.

* Locales.

* autoload_paths 下的文件.

* db/schema.rb 和 db/structure.sql.

当其中的任何文件发生改变时， rails将会调用`Module#remove_const`方法，移除所有自动加载的常量，并重新加载这些常量的定义。


