---
layout: post
title:  "one definition rule"
date:   2015-07-02 13:17:58
categories: jekyll update
---
昨天碰到一个棘手的问题，编译没有出错，只有几个warning，在运行的时候却挂掉，报段错误。怀疑是空指针问题，但看代码实在不存在空指针的情况。后来不经意改了个变量名，却神奇的可以运行。经过我不懈查找，终于找到原因，这个错误正是前面warning暗示的问题。这个故事再次告诫我们，warning绝对不能不管，要尽一切可能排除warning。

这个问题我在stackoverflow上进行了[提问][]，才知道是违反了[One Definition Rule]。可以得到这样的结论，对于同一个对象（只要名字相同就认为是同一个对象），不允许它有两个定义，无论这两个定义是否有相同的类型。如果这两个定义在同一个文件中，那么编译都不能通过，如果跨越文件，编译器不负责检查，这样在运行时的行为就是不确定的，得到段错误也不足为奇。

同时，如果必须要跨越文件使用同一个对象，那么应该以[这样]的方式使用extern。而如果只是在文件内部使用又想用同一个名字的话，可以给他们加static，这样就保证了这个变量是限定在文件范围内的。

[提问]: http://stackoverflow.com/questions/31180474/core-dumped-when-a-function-pointer-is-assigned-with-a-funtion-that-has-the-same
[这样]: http://stackoverflow.com/questions/1433204/how-do-i-use-extern-to-share-variables-between-source-files-in-c/1433387#1433387
[One Definition Rule]: https://en.wikipedia.org/wiki/One_Definition_Rule
