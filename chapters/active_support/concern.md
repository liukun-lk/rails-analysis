# ActiveSupport::Concern

### 作用
1. 让 Ruby 类包含模块之后同时获得实例方法和类方法；
2. 简化 module mix 的依赖问题。

### 使用介绍
##### 1. 让 Ruby 类包含模块之后同时获得实例方法和类方法

原本 Ruby 自带了`include/extend/prepend`三个方法，我们可以通过这三个方法扩展类的方法。具体可以参考 [Ruby 中 require,load,autoload,extend,include,prepend 的区别](https://ruby-china.org/topics/35350)

一般在扩展实例方法的时候需要使用`include`，而扩展类方法的时候需要使用`extend`，我们就会想如何能把扩展写到一个 module 来做？所以就有人使用 `included` 的钩子来实现，传统做法是：
```ruby
module M
  def self.included(base)
    base.extend ClassMethods
    base.class_eval do
      scope :disabled, -> { where(disabled: true) }
    end
  end

  module ClassMethods
    ...
  end
end
```

一旦你使用 `ActiveSupport::Concern`，只需要
```ruby
require 'active_support/concern'

module M
  extend ActiveSupport::Concern

  included do
    scope :disabled, -> { where(disabled: true) }
  end

  class_methods do
    ...
  end
end
```
但是自己通过`included`钩子实现的代码看上去其实很简单，为什么还需要写一个库？这就引出了另一个问题。

先看以下代码：
```ruby
module Foo
  def self.included(base)
    base.class_eval do
      def self.method_injected_by_foo
        ...
      end
    end
  end
end

module Bar
  include Foo
  def self.included(base)
    base.method_injected_by_foo
    # base.include Foo 如果加上这行，则可以删去下面 include Foo
  end
end

class Host
  include Foo # We need to include this dependency for Bar
  include Bar # Bar is the module that Host really needs
end
```

如果在 `Host` 类里面不加 `include Foo` 的话会报 ```NoMethodError: undefined method `method_injected_by_foo' for Host:Class```。很明显每次 `include Foo` 之后还需要加 `base.include Foo`，不仅代码看上去比较冗余，而且也会给后面维护代码的人造成影响。如果他并不清楚这边存在的问题，很可能就会造成一个 bug。

##### 2. 简化 module mix 的依赖问题

为了防止上面的问题，所以 Rails 引入了 `ActiveSupport::Concern`，通过 git history 可以明显的看到，该类是从一个叫做 `ActiveSupport::DependencyModule` 分出来的，所以早期可能就是为了解决 module 的依赖问题而开发的这个工具类。[后面](https://github.com/rails/rails/commit/4e50a35fa243f6cf7ad567774a9f7c1cb87a1653#diff-0e2f2ed89c021f5f97a943c01061476b)才从 `ActiveSupport::DependencyModule` 类中分出。不过最初的 `ActiveSupport::Concern` 只具备「作用1」的功能，依赖关系还是由 `ActiveSupport::DependencyModule::depends_on` 来控制，直到 [commit: Refactor AS concern to avoid hacking the "include" method.](https://github.com/rails/rails/commit/7ec947d59c1bc3e9772788b757fe70f51b0ffd9b#diff-94b4d6e0912ace89981a461b6b914c8b) 才彻底将 `ActiveSupport::DependencyModule` 中的依赖问题移到 `ActiveSupport::Concern` 中去解决。


### 源码阅读

```ruby
module Concern
  class MultipleIncludedBlocks < StandardError #:nodoc:
    def initialize
      super "Cannot define multiple 'included' blocks for a Concern"
    end
  end

  def self.extended(base) #:nodoc:
    base.instance_variable_set(:@_dependencies, [])
  end

  def append_features(base)
    if base.instance_variable_defined?(:@_dependencies)
      base.instance_variable_get(:@_dependencies) << self
      false
    else
      return false if base < self
      @_dependencies.each { |dep| base.include(dep) }
      super
      base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
      base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)
    end
  end
# ...
```

上面主要是 `extend` 和 `append_features` 两个方法。

当模块扩展 Concern 的时候，会调用钩子 `extended`。在这个方法中为扩展它的类定义一个 `@_dependencies` 实例对象且赋值为 `[]`。
```ruby
module A
  extend ActiveSupport::Concern
  # ...
end
# 其中 A 就会赋值给 Concern 模块中的 base。当 extend ActiveSupport::Concern 后 A 中就会存在一个 @_dependencies 为 [] 的对象。
```
方法 `append_features` 是运行于 `included` 之前的，如果其原本功能被覆盖掉的话，模块的引入就失效了。`append_features` 方法包含实际动作，用于检查被包含模块是否已经在包含类的祖先链上，如果不在，则将该模块加入其祖先链，而 `included` 默认实现是空的，只有覆写后才有内容。

---

### 流程图

```ruby
module C
  extend ActiveSupport::Concern

  included do
    # 此处不再需要使用 base，直接使用 self 关键字引用上下文即可
    self.greeting = "welcome, buddy!"
  end
end

module B
  extend ActiveSupport::Concern

  include C
end

class A
  class << self
    attr_accessor :greeting
  end

  # 只需混入 B
  include B
end
```

![concern 流程图](/assets/images/activesupport_concern.png)

---

### 有趣的 commit

1. [Use symbols instead of strings in ActiveSupport::Concern](https://github.com/rails/rails/pull/10909)
该 PR 是将 `ActiveSupport::Concern` 内部的字符串都换成 symbol 来防止对象的再创建，最后有相关的 benchmark。

2. [Use public Module#include, in favor of https://bugs.ruby-lang.org/issues/8846](https://github.com/rails/rails/pull/18767)
该 PR 是 Ruby 2.1.0 将原本私有的 `include` 方法变成公有方法。

3. [ActiveSupport::Concern#class_methods affects parent classes](https://github.com/rails/rails/issues/20489)
该 issue 提到一种特殊案例：

```ruby
require 'active_support/concern'

module Foo
  extend ActiveSupport::Concern

  class_methods do
    def foo

    end
  end
end

module Bar
  extend ActiveSupport::Concern

  class_methods do
    def bar

    end
  end
end

class Parent
  include Bar
end

class Child < Parent
  include Foo
end

Parent.respond_to? :foo # returns true
```
按理来说只是在子类 `include Foo`，父类不应该会包含 `foo` 的类方法。所以需要在判断 `const_defined?(sym, inherit=true)` 时添加第二个参数来表示只在当前类/模型查看 `ClassMethods` 这个 module 是否定义。

---

### 课外概念 - class_eval and instance_eval
##### class_eval 打开类
首先 `class_eval` 的调用者（receiver）必须是一个类（因为 Ruby 中类也是一个对象，也可以被当做一个对象调用），而在 `class_eval block` 的内部，`self` 即为 `receiver` 类本身。

```ruby
class A
end

A.class_eval do
  def foo
    ...
  end
end

# 可以等同于在 A 类中定义了一个 foo 的方法，自然该 foo 方法为类 A 的实例方法。
class A
  def foo
    ...
  end
end
```

##### instance_eval 打开实例对象
`instance_eval` 的调用者（receiver）必须是一个实例，而在 `instance_eval block` 的内部，`self` 即为 `receiver` 实例本身。
根据这个定义，如果在一个实例上调用了 `instance_eval`，就可以在其中定义该实例的单态函数 `singleton_method`。

```ruby
class A
end

a = A.new
a.instance_eval do
  puts self  # => a
  # current class => a's singleton class
  def method1
    puts "this is a singleton method of instance a"
  end
end

a.method1
#=> this is a singleton method of instance a

b = A.new
b.method1
#=> NoMethodError: undefined method `method1' for #<A:0x007fbc2ced9550>
from (pry):13:in `<main>'
```

当然上面有说到 Ruby 中的 class 其实也是一个实例对象，如果在 class 上面调用 `instance_eval`，相当于给 class 定义了一个单例方法，而 class 的单例方法其实也就是类方法。
```ruby
class A
end

A.instance_eval do
  puts self  # => A
  # current class => A's singleton class
  def method1
    puts 'this is a singleton method of class A'
  end
end

A.method1
# this is a singleton method of class A
#=>  nil
```

### 参考资料
* [理解 ActiveSupport::Concern](https://ruby-china.org/topics/26208)
* [Active Support 的 Concern 模块来由探究](https://ruby-china.org/topics/32585)
* [Concern](https://book.a-bitcoin.com/activesupport/activesupport_concern.html)
* [ActiveSupport::Concern 的工作原理](http://xiewenwei.github.io/blog/2014/01/13/how-activesupport-corncern-work/)
* [History for rails/activesupport/lib/active_support/concern.rb](https://github.com/rails/rails/commits/master/activesupport/lib/active_support/concern.rb)
* [浅析Module类中included和append_features两个回调方法的区别](http://elfxp.com/different-between-included-and-append_features)
* [rails源码赏析之Concern](http://elfxp.com/intro-of-concerns-in-rails)
* 《Ruby 元编程（第2版）》