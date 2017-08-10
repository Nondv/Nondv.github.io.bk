---

title:  'Postgresql, неравенство и NULL'
categories: development
tags: [rails, postgresql, notes]
---

## Intro

Постоянно забываю о том, что неравенство в Postgresql при сравнении с
NULL ведет себя не совсем очевидным (на мой взгляд) образом.

<!--more-->

## Суть

Если в операторах сравнения *хотя бы один* из операндов будет NULL,
то и результатом сравнения будет NULL.

Это означает, что `1 < NULL`, `NULL <> NULL`, `NULL = NULL` будут
вычисляться, как NULL (особенно странным это выглядит для равенства,
но это является SQL-стандартом, судя по документации постгреса).

Недавно писал скоуп в рельсах, который выглядел так:

```ruby
scope :public_templates, -> { where.not(personal: true) }
```

И тесты, в которых использовались записи, в которых флаг `personal`
не был установлен (NULL), падали. Все потому, что это условие
превращается в `personal <> TRUE`, что не срабатывает для NULL.

Вот так вот. Напрашивающееся красивое решение не работает.

## Решение

В случае постгреса есть т.н. null-safe оператор равенства:

```
IS [NOT] DISTINCT FROM
```

В моем случае его использование выглядело так:

```ruby
scope :public_templates, -> { where('templates.personal IS DISTINCT FROM TRUE') }
```

Для ознакомления:
https://www.postgresql.org/docs/current/static/functions-comparison.html
