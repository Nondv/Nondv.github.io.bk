---
layout: page
title: rubocop-config
permalink: /rubocop-config.html
---


Я решил выложить его для того, чтобы не копипастить конфиги между проектами, а
просто использовать `inherit_from`.

Таким образом, я могу вносить правки в свою конфигурацию независимо от
конкретного проекта.

[raw](/rubocop-config.yml)

```yml
{% include_relative rubocop-config.yml %}
```
