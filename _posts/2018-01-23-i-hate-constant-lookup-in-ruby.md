---

title:  'Я ненавижу поиск констант в руби'
categories: development
tags: [ruby, notes]
---

Я люблю Ruby. Но некоторые вещи в нем мне кажутся крайне странными. Вплоть до
ненависти (например, оператор `case`). В данном посте я просто приведу кусок
кода, который вызывает у меня ненависть к Ruby.

На хабре я написал об этом более подробно:
[https://habrahabr.ru/post/347272/](https://habrahabr.ru/post/347272/)

<!--more-->

```ruby
module M
  A = 'm'
end

module Namespace
  A = 'ns'
  class C
    include M
  end
end

puts Namespace::C::A # m

module Namespace
  class C
    puts A # ns
  end
end

module M
  def m
    A
  end
end

module Namespace
  class C
    def f
      A
    end
  end
end

class Namespace::C
  def g
    A
  end
end

x = Namespace::C.new
puts x.f # ns
puts x.g # m
puts x.m # m
```
