# SpannableStringBuilder

`SpannedString`, `SpannableString`, `SpannableStringBuilder`

区别

`SpannedString` 和 `SpannableString` 使用数组

`SpannableStringBuilder` 使用 `interval tree`

使用场景

Class                  | Mutable Text | Mutable Markup
:--------------------- | :----------- | :-------------
SpannedString          | no           | no
SpannableString        | no           | YES
SpannableStringBuilder | YES          | YES

## 参考

- [Spantastic text styling with Spans](https://medium.com/androiddevelopers/spantastic-text-styling-with-spans-17b0c16b4568)
- [Underspanding spans](https://medium.com/androiddevelopers/underspanding-spans-1b91008b97e4)
- [线段树从零开始](https://blog.csdn.net/zearot/article/details/52280189)
