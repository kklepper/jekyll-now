---
layout: page
title: About
permalink: /about/
---

Some information about you!

### More Information

A place to include any other types of information that you'd like to include about yourself.

### Contact me

[email@domain.com](mailto:email@domain.com)









# Table of Contents

1. [Workaround for shells without LINENO](#workaround for shells without LINENO)
2. [Debugging](#debugging)



## Workaround for shells without LINENO

Lucky are those who have bash, but ash does not have LINENO, alas. Which is bad if you need to do a lot of testing on `Busybox` or the like.

That's why I finally developed a simple and powerful workaround with interesting side effects for debugging and logging purposes. Instead of using  or `Boot2Docker` or `Tiny Linux`, introduce a small and simple function `echo_line_no` (or whatever function name you like better) which prints the line number pretty much, but not exactly like LINENO. 

Use it with quotes like isline in the source file is 16. Fine.



Sure. Why do you need this? How do you work with this? What can you do with this? Why do you want to tinker with this at all?

## Debugging

Line numbers are not just for fun, they are badly needed for debugging, and that's where things get more complicated. 

I could just leave all of that as an exercise to you, but this is not the intention of Stack Overflow as I understand it. Hence I elaborate on the subject to give you as complete an insight as is possible for me. 

Sorry, it gets a bit lengthy for that reason.

In case you fear I get talkative or be prating, just stop reading. 

In particular I don't want to insult all you experts who are better than I and can do all of this on their own. 

It's for people like me I take the pain to write this all up, those who are new to the subject and have to fight their way more or less alone. Kind of paying back my debt according to the old mailing list ethics. Bear in mind that I can only talk from my own experience, which is limited. So take this text with a grain of salt.

Back to work. For example, if you like to use multi-line comments or show variables in your debugging messages, that simple approach given above will not work for those cases. 