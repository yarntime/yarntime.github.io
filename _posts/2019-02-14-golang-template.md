# 使用golang template生成代码时遇到的问题

###问题：

```
{{- $auth := ""}}
{{- if not $element.Insecurity}}
{{- $auth = ", nil"}}
{{- end}}

```
如上代码片段所示，需要根据条件设置auth变量的值。

使用template解析时报错：

```
unexpected "=" in operand
```

###问题原因：
golang早期版本的template设计中没有=或者赋值运算符。

golang1.11版本以后支持=号赋值。

###解决方案：
升级golang到1.11+版本