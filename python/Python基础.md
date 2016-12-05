# [字符编码](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001431664106267f12e9bef7ee14cf6a8776a479bdec9b9000)

- ASCLL 最先出现的编码，只支持英文，占1个字节

- GB2312 ASCLL1个字节显然不能涵括汉字，所以，中国制定了GB2312编码，用来把中文编进去，至少需要两个字节，同理世界上的文字种类众多，会有好多各国定制编码，但同时需要兼容ASCLL编码

- Unicode(Universal Character Set) 通用编码，把所有语言都统一到一套编码里，占用两个字节（或四个字节）

- Utf-8(Universal Character Set Transformation Format ) Unicode的固定长度会造成空间的浪费，UTF-8编码把一个Unicode字符根据不同的数字大小编码成1-6个字节，所以ASCII编码实际上可以被看成是UTF-8编码的一部分
