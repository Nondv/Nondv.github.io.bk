---

title:  'Типы методов в Ruby: private vs protected'
categories: development
tags: [ruby, notes]
---

## Intro

В Ruby очень вычурно используется модификатор `protected`.
Лично я считаю бесполезной практикой его использование, но, знать, как
он работает, может быть полезно (на собеседовании, например).

Кстати, `private` тоже имеет кое-какую особенность;)

<!--more-->

## Private

Приватные методы, как и в других языках, могут быть вызваны только
в контексте (внутри) объекта.

Причем в Ruby правило формулируется примерно так: приватный метод
нельзя вызвать на объекте явно (в т.ч. на `self`), он может быть вызван
только неявно на `self` экземплярами класса или его подклассов.

```ruby
class C
  def m
    self.private_m
  end

  private

  def private_m
    puts "no you don't"
  end
end

C.new.m # ==> error
```

Если совсем коротко: приватные методы не могут быть вызваны на объекте
*явно* (через точку).


## Protected

Сначала процитирую определение из книги The Ruby Programming Language:

> a protected method defined by a class C may be invoked on an object
> o by a method in an object p if and only if the classes of o and p
> are both subclasses of, or equal to, the class C.

В общем, protected  - это что-то среднее между public и private.
Его можно вызвать явно, но только из контекста (метода, например)
объекта, который является экземпляром того же класса или его подкласса.

Пример:

```ruby
class C
  protected

  def protected_m
    'hello'
  end
end

class D < C
  def m
    C.new.protected_m
  end
end

D.new.m # ==> 'hello'
C.new.protected_m # ==> error
```


## Заключение

Я не могу придумать адекватной причины, по которой стоит использовать
`protected` вместо `private`. Разве что если вы любитель вызывать
методы на `self` явным образом (protected методы можно вызвать так, в
отличие от private).

Абсолютно бесполезная фича языка. Но вдруг кто-то спросит...
