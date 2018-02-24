---
layout: post
title: You're up and running!
---

Next you can update your site name, avatar and other options using the _config.yml file in the root of your repository (shown below).

![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.



# Table of Contents

1. [Workaround for shells without LINENO](#workaround)
2. [Debugging](#debugging)



## Workaround for shells without LINENO

Lucky are those who have bash, but ash does not have LINENO, alas. Which is bad if you need to do a lot of testing on `Busybox` or the like.

That's why I finally developed a simple and powerful workaround with interesting side effects for debugging and logging purposes. Instead of using 

```
echo $LINENO this is a simple comment with a line number
```

which does not work with e.g. `Busybox` or `Boot2Docker` or `Tiny Linux`, introduce a small and simple function `echo_line_no` (or whatever function name you like better) which prints the line number pretty much, but not exactly like LINENO. 

Use it with quotes like 

```
echo_line_no "this is a simple comment with a line number"
```

Output is

```
16   "this is a simple comment with a line number"
```

if the number of this line in the source file is 16. Fine.

The definition of this function is:

```
echo_line_no () {
    cat -n $0 | grep "$1" |  sed "s/echo_line_no//g" 
    # show the file content with line numbers
    # grep the line(s) containing $input
    # replace the string echo_line_no with nothing 
} # echo_line_no
```

This basically answers the question [How to show line number when executing bash script](https://stackoverflow.com/questions/17804007/how-to-show-line-number-when-executing-bash-script). Anything more to add? 

Sure. Why do you need this? How do you work with this? What can you do with this? Why do you want to tinker with this at all?

## Debugging

Line numbers are not just for fun, they are badly needed for debugging, and that's where things get more complicated. 

I could just leave all of that as an exercise to you, but this is not the intention of Stack Overflow as I understand it. Hence I elaborate on the subject to give you as complete an insight as is possible for me. 

Sorry, it gets a bit lengthy for that reason.

In case you fear I get talkative or be prating, just stop reading. 

In particular I don't want to insult all you experts who are better than I and can do all of this on their own. 

It's for people like me I take the pain to write this all up, those who are new to the subject and have to fight their way more or less alone. Kind of paying back my debt according to the old mailing list ethics. Bear in mind that I can only talk from my own experience, which is limited. So take this text with a grain of salt.

Back to work. For example, if you like to use multi-line comments or show variables in your debugging messages, that simple approach given above will not work for those cases. 