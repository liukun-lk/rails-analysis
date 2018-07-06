# ActiveSupport::Autoload

### Ruby 包管理历史

![Ruby require 历史](/assets/images/require.png)
很幸运的是，当笔者接触 Ruby 这门语言的时候，Ruby 已经使用 Bundler 这个工具来解决依赖库的管理。如果有对这个历史感兴趣的，可以看[Bundler 到底是怎么工作的 (暨 Ruby 依赖管理历史回顾)](https://ruby-china.org/topics/28453)，里面很详细的介绍了进化的过程。

### Ruby 加载代码

一般我们写代码的时候，除了那些简单的程序，大多数情况我们不会将所有的代码写到一个文件里，而是根据不同的功能拆分成多个文件。
假如我们在一个文件夹内有以下两个文件：
```ruby
# codes/person.rb
class Person
  def greet
    puts "Hello."
  end
end

# codes/main.rb
person = Person.new
person.greet
```
在 main.rb 中我们并不能直接使用 person.rb 中的代码，需要先把 person.rb 中的代码 require 进来。
```ruby:codes/main.rb
require './person'
```
这是最基本的 require 用法，直接加上相对路径，但是使用相对路径时，默认的当前路径并不是固定不变的，而绝对路径的话，可能换个环境就要做出相应变化。因此 Ruby 给出了两个全局变量 `$LOADED_FEATURES(aka $")，$LOAD_PATH(aka $:)`。其中，`$LOADED_FEATURES` 里保存了已经 require 过的文件，`$LOAD_PATH` 里保存了查找文件的路径，两者都是数组结构。

以 `require 'benchmark'` 为例，典型的查找流程是这样，先在 `$LOADED_FEATURES` 里查找是否已经 require 过这个文件，如果已经 require 过，则返回 false，如果没有，则会在 `$LOAD_PATH` 内依次查找这个文件。

因此上面的例子就可以改写成：
```ruby
# codes/main.rb
require 'person'

# codes/scripts/run.rb
require 'main'
```
其实我们经常可以在 Gem 源码中的 gemspec 里看到：
```ruby
lib = File.expand_path('../lib', __FILE__)
$LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib)
```
这句话的意思就是在打包 Gem 的时候把 lib 添加到 `$LOAD_PATH` 内。

不过和 require 类似的，还有 load 和 autoload 两个方法。这里介绍一下 autoload，autoload 是 Ruby 中实现懒加载的方法。

当我们使用 autoload 的时候，仅仅是注册了一个常量，真正的代码并没有被 require 进来，只有当我们第一次使用这个常量的时候，才会去 require 这个注册的文件。以一开始的代码为例：
```ruby:codes/main.rb
autoload :Person, './person.rb'
puts "LOADED_FEATURES: #{$LOADED_FEATURES.select {|item| item.index('person')}}" # => LOADED_FEATURES: []
p = Person.new
puts "LOADED_FEATURES: #{$LOADED_FEATURES.select {|item| item.index('person')}}" # => LOADED_FEATURES: ["/path/demo/person.rb"]
```
当代码加入到 `$LOADED_FEATURES` 时，才是真正被加载进来。而 `$LOAD_PATH` 只是管理了项目可查找的文件路径罢了。

##### 此 require 非彼 require

实际上上面的 require 说明的情况仅仅适用于最基本的情况，require 是 `Kernel.require` 的方法，而实际上我们常常使用的不是这个方法。
```ruby
# 打开irb输入
method(:require).source_location
# => ["/Users/feelmix/.rvm/rubies/ruby-2.4.0/lib/ruby/site_ruby/2.4.0/rubygems/core_ext/kernel_require.rb", 39]
```
因为 rubygems 已经重写了 require 方法，并且 rubygems 现在已经随着 Ruby 一起下发，所以我们使用的都是这个方法。

因此在使用这个 require 的时候，当 `$LOAD_PATH` 内找不到的这个文件，就会使用 `Gem.find_files 'file'` 来查找，如果有结果就会 require 相应的文件。

我们常常在 Gem 的 gemspec 里找到如下代码：
```ruby
Gem::Specification.new do |spec|
  ...
  spec.require_paths = ["lib"]
end
```
而且我们会发现，通常情况下我们在写 Rails 项目的时候并不需要 require Gemfile 里面的 Gem，是因为我们在启动项目的时候执行了下面的方法：
```ruby
require 'bundler/setup'
Bundler.require(*Rails.groups)
```
`Bundler.require(*Rails.groups)` 会将指定版本的 Gem require 到项目中来。

### ActiveSupport::Autoload 作用

1. 根据用户定义的类名猜出文件路径；
2. 设置延迟加载(自动加载)和预加载(立即加载)。

### 源码阅读

##### 根据用户定义的类名猜出文件路径

当我们写一个 Gem 的时候，常常会为了代码的组织去更改文件的位置，修改文件的命名空间(文件夹)，如果使用了 require 或者 autoload 方法，就会需要经常性的去修改相应的目录位置。而使用 `ActiveSupport::Autoload`，用户只需要定义：
```ruby
module MyLib
  extend ActiveSupport::Autoload

  autoload :Model

  eager_autoload do
    autoload :Cache
  end
end
```
`ActiveSupport::Autoload` 就会自动猜出 `Model` 和 `Cache` 所在的文件路径，并将文件加载到项目中。

关键代码如下：
```ruby:rails/activesupport/lib/active_support/dependencies/autoload.rb#L37
def autoload(const_name, path = @_at_path)
  unless path
    full = [name, @_under_path, const_name.to_s].compact.join("::")
    path = Inflector.underscore(full)
  end
  ...
end
```
`ActiveSupport::Autoload` 覆写了 `autoload` 方法，主要是将传入的 path 参数设为了可选，在后面调用 Ruby 的 autoload 方法时，Rails 会自动配置 path。

其实理解了 extended 钩子方法所定义的四个实例变量，也就理解了 `ActiveSupport::Autoload` 的用法：
```ruby
def self.extended(base) # :nodoc:
  base.class_eval do
    # 为了后面 eager_load! 时将代码加载进来
    @_autoloads = {}
    # 为了可以手动配置文件夹路径，因为有些文件未必按照约定的文件夹来放置
    @_under_path = nil
    # 直接手动设置文件路径，设置了该变量相当于直接使用 Ruby 的 autoload 方法
    @_at_path = nil
    # 默认情况下都是使用延迟加载
    @_eager_autoload = false
  end
end
```

##### 设置延迟加载(自动加载)和预加载(立即加载)

`ActiveSupport::Autoload` 内部主要通过 `autoload` 和 `eager_autoload` 分别实现延迟加载(自动加载)和预加载(立即加载)。

autoload 可以分别被 autoload_under, autoload_at 和 eager_autoload 嵌套，前两个用来自动配置 path 使用，后面的 eager_autoload 用来给需要立即加载的代码所使用。使用方法可参考 [Reorganize autoloads](https://github.com/rails/rails/commit/c1304098cca8a9247a9ad1461a1a343354650843#diff-4b071416c9f83e7120394f41d75cdd27)。

### 有趣的 commit

1. [Flip deferrable autoload convention](https://github.com/rails/rails/commit/ace20bd25e3818b7f29c222643dd445c48b36425#diff-4b071416c9f83e7120394f41d75cdd27)
最开始的 `ActiveSupport::Autoload` 是将 Ruby 的 autoload 方法覆写成 require 加载(立即加载)，在该 commit 中转换了加载方式。

2. [Improved ActiveSupport::Autoload performance.](https://github.com/rails/rails/commit/6b480d2e8260b88474a33f1b45847e0ad8b1bc96#diff-4b071416c9f83e7120394f41d75cdd27)
该 commit 只是在每次自动生成 path 的时候先判断一下是否已传入 path，就这样简单的修改性能提高了三倍。

3. [Using require_relative in the Rails codebase](https://github.com/rails/rails/pull/29638)
该 commit 最开始的目的只是因为 `Kernel#require_relative` 没有被第三方的 gem 重写过，因此性能上会比 require 好一些。但也因为这次修改，突然发现了 Ruby 中的一个 bug：[require_relative and require should be compatible with each other when symlinks are used](https://bugs.ruby-lang.org/issues/10222).
你在小于 Ruby 2.5 的版本上面跑时，以下命令最后会输出两次 "b loaded"
```ruby
mkdir a
ln -s a b
echo "require_relative 'b'" > a/a.rb
echo "p 'b loaded'" > a/b.rb
echo "$: << File.expand_path('../b', __FILE__); require 'a'; require 'b'" > c.rb
ruby c.rb
```
不过后面修复了 `require_relative`，只让该方法存放真实的路径，而不是符号链接。

### 参考资料
* [Ruby 包管理](https://zhongfox.github.io/2016/10/25/ruby-gem-management/)
* [Ruby加载代码的二三事](http://blog.jerry-tao.com/2016/01/12/load_code_in_ruby/)
* [Ruby 語法放大鏡之「你知道 require 幫你做了什麼事嗎?」](https://kaochenlong.com/2016/05/01/require/)
* [Rails 中的类加载机制](https://ruby-china.org/topics/26034)
* [Ruby Hacking Guide-Chapter 18: Loading](http://ruby-hacking-guide.github.io/load.html)
* [How to use Bundler with Ruby](https://bundler.io/v1.16/guides/bundler_setup.html)
* [Rails 源码分析之 eager_load! 篇](https://ruby-china.org/topics/26866)
* [ActiveSupport::Autoload 学习](https://ruby-china.org/topics/25021)
* [Eager loading for greater good](http://blog.plataformatec.com.br/2012/08/eager-loading-for-greater-good/)
