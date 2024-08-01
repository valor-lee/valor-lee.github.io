---
title: '[Shell] 문자 수정 편리한 기능' 
date: 2024-08-01 18:29:00 +09:00
categories: [language, shell]
tags:
  [
    shell
  ]
---


# 문자 치환

## `${variable//pattern/replacement}`

- 변수에 저장된 문자열에서 모든 pattern을 replacement로 치환하는 역할


```shell
list="a, b, c, d"

list=${list//,/ }
echo $list

# >> a b c d
```


---

참조
