### Rails 中出现过的概念
- Active Model（2.0stable出现
- Action Pack
- Active Record
- Active Resource(2.0stable出现 4.0stable消失？4.2stable又出现？)
- Active Support
- Action Mailer
- Action View(4.2stable出现)
- Active Job(4.2stable出现)
- Action Cable(5.0stable出现)
- Active Storage(5.2stable出现)
- Action Web Service(1.2stable出现 2.0stable消失)
- railties

### 文章中还未讲到的内容
##### eager_load! 实际上分了2派：
1. 通过 activesupport/lib/active_support/dependencies/autoload.rb 提供的 eager_load!, 也就是去 extend ActiveSupport::Autoload。针对于非engine的其它namespace
2. 基于 engine 的 eager_load! 的实现，需要去设置变量 eager_load_paths。railties/lib/rails/engine/configuration.rb 有设置

##### Rails 中的类加载机制
- https://ruby-china.org/topics/26034
- [Rails autoloading — how it works, and when it doesn't](https://urbanautomaton.com/blog/2013/08/27/rails-autoloading-hell/)

##### 弃用 autoload
- https://bugs.ruby-lang.org/issues/5653
- https://bugs.ruby-lang.org/issues/5654
- https://bugs.ruby-lang.org/issues/921
- [Do we embrace Kernel#autoload ? ](https://github.com/ruby-concurrency/concurrent-ruby/issues/395)