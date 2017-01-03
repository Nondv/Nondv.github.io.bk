---

title:  'Парсинг опций в Ruby'
categories: development
tags: [ruby, console, cli, notes]
---

# Intro

Пару месяцев назад узнал, что в стандартной библиотеке Ruby есть
класс `OptionParser`, позволяющий обрабатывать `ARGV` на предмет
всяких флажков. Решил сделать заметку, чтобы не забыть.

<!--more-->

# Доки

[Вот](http://ruby-doc.org/stdlib-2.4.0/libdoc/optparse/rdoc/OptionParser.html).

Ссылка на версию Ruby 2.4.0, но класс есть и на маковской версии 2.0.0:

```
$ /usr/bin/ruby -v
ruby 2.0.0p648 (2015-12-16 revision 53162) [universal.x86_64-darwin16]
```

# Заметки

Вот кусок кода из моего скрипта `cal` (выкинул на [гитхаб](https://github.com/Nondv/cal)):

```ruby
OptionParser.new do |opts|
  opts.banner = 'Usage: cal [options] [dd.mm.yyyy]'

  opts.on('-v', '--version', 'Prints version') do
    puts "cal v#{PrettyCalendar::VERSION}"
    exit
  end

  opts.on('-h', '--help', 'Prints this help') do
    puts opts
    exit
  end
end.parse!

if ARGV.empty?
  puts PrettyCalendar.today
else
  puts PrettyCalendar.parse(ARGV.first)
end

```

### Получение аргументов, не относящихся к флажкам

Во время обработки `ARGV` с помощью `parse!` парсер удаляет оттуда
распарсенные опции (так что не стоит пытаться потом обрабатывать их
повторно!), поэтому для того, чтобы узнать, что "осталось" просто
чекаем `ARGV`. См конец примера кода выше.

### Получение аргумента опции

Изначально я хотел сделать не `cal 01.01.2017`, а `cal -d 01.01.2017`.
Соответственно, парсеру нужно дать понять, что после `-d` будет идти
*обязательный* аргумент. Делается это просто с помощью аргументов
`opts.on`:

```
opts.on('-d DATE`, `--date DATE`, `Specifies date`) do |date_string|
  ...
end
```

### Приведение типов

`OptionParser` можно задавать типы аргументов для опций. Не буду
описывать это, т.к. сам не пользовался. Подмечу лишь то, что есть
набор классов, к которым он может приводить из коробки, но можно
добавить и кастомные.

[Вот ссылка на документацию](http://ruby-doc.org/stdlib-2.4.0/libdoc/optparse/rdoc/OptionParser.html#class-OptionParser-label-Type+Coercion)
