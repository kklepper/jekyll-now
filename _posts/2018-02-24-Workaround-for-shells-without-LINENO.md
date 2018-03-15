---
layout: post
title: Workaround for shells without LINENO
published: true
---

... and some other reflections
----------

## Table of Content <span id="toc"></span>

- [Workaround for shells without LINENO](#workaround-for-shells-without-lineno-table-of-content)
- [Debugging](#debugging-table-of-content)
- [More complex cases](#more-complex-cases-table-of-content)
- [How to use](#how-to-use-table-of-content)
- [Caveats](#caveats-table-of-content)
- [Logging](#logging-table-of-content)
- [Example](#example-table-of-content)
- [Why roll your own?](#why-roll-your-own-table-of-content)
- [Database replication error handling](#database-replication-error-handling-table-of-content)
> - [Digression: Repair or copy](#digression-repair-or-copy-table-of-content)
> - [Digression: Table partitioning](#digression-table-partitioning-table-of-content)
> - [Digression: Partition by md5](#digression-partition-by-md5-table-of-content)
> - [Digression: PRIMARY KEY clause](#digression-primary-key-clause-table-of-content)
> - [Digression: Unusable partition distribution](#digression-unusable-partition-distribution-table-of-content)
> - [Digression: Experimenting with CONV](#digression-experimenting-with-conv-table-of-content)
> - [Digression: Max value of bigint datatype](#digression-max-value-of-bigint-datatype-table-of-content)
> - [Digression: Table type: MyISAM vs. InnoDB](#digression-table-type-myisam-vs-innodb-table-of-content)
- [Database replication regular health checking](#database-replication-regular-health-checking-table-of-content)
- [Database replication automatic failover](#database-replication-automatic-failover-table-of-content)
- [Adding a stopwatch](#adding-a-stopwatch-table-of-content)
> - [Digression: Docker and mysqlbinlog](#digression-docker-and-mysqlbinlog-table-of-content)
> - [Digression: Partitioning by RANGE](#digression-partitioning-by-range-table-of-content)
> - [Digression: Reorganizing sql logging](#digression-reorganizing-sql-logging-table-of-content)
> - [Digression: Comments and editors](#digression-comments-and-editors-table-of-content)
> - [Digression: Other tools](#digression-other-tools-table-of-content)
> - [Digression: Speech recognition](#digression-speech-recognition-table-of-content)
> - [Digression: WSR vs. DragonDictate](#digression-wsr-vs-dragondictate-table-of-content)
> - [Digression: Dictation workflow](#digression-dictation-workflow-table-of-content)
> - [Digression: AHK magic](#digression-ahk-magic-table-of-content)
> - [Digression: Key trouble](#digression-key-trouble-table-of-content)
> - [Digression: Explanation, no solution](#digression-explanation-no-solution-table-of-content)
> - [Digression: Pronunciation](#digression-pronunciation-table-of-content)
> - [Digression: Hello computer](#digression-hello-computer-table-of-content)
> - [Digression: Partitioning by day](#digression-partitioning-by-day-table-of-content)
> - [Digression: Partitioning by day of week](#digression-partitioning-by-day-of-week-table-of-content)
> - [Digression: Comparing files](#digression-comparing-files-table-of-content)
> - [Digression: Why are data files different](#digression-why-are-data-files-different-table-of-content)
> - [Digression: Automatic git checkout](#digression-automatic-git-checkout-table-of-content)
> - [Digression: Automatic boot2docker setup](#digression-automatic-boot2docker-setup-table-of-content)
- [Why roll your own, revisited](#why-roll-your-own-revisited-table-of-content)
- [Have fun](#have-fun-table-of-content)
> - [Digression: Proof of concept](#digression-proof-of-concept-table-of-content)
> - [Digression: Record by database](#digression-record-by-database-table-of-content)
> - [Digression: More complexity by languages](#digression-more-complexity-by-languages-table-of-content)
> - [Digression: Dirty debugging techniques](#digression-dirty-debugging-techniques-table-of-content)
> - [Digression: Dirty example](#digression-dirty-example-table-of-content)
> - [Digression: Adding microtime by trigger](#digression-adding-microtime-by-trigger-table-of-content)
> - [Digression: Adding microtime natively](#digression-adding-microtime-natively-table-of-content)
> - [Digression: Language versions](#digression-language-versions-table-of-content)
> - [Digression: Analyzing data](#digression-analyzing-data-table-of-content)
> - [Digression: Adding a stopwatch by database](#adding-a-stopwatch-by-database-table-of-content)
> - [Digression: Erlang style](#digression-erlang-style-table-of-content)
- [Search engines](#search-engines-table-of-content)
- [A big thank you to you all](#a-big-thank-you-to-you-all-table-of-content)

Workaround for shells without LINENO <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Define a function

    echo_line_no () {
        grep -n "$1" $0 |  sed "s/echo_line_no//" 
        # grep the line(s) containing input $1 with line numbers
        # replace the function name with nothing 
    } # echo_line_no

Use it with quotes like 

    echo_line_no "this is a simple comment with a line number"

Output is

    16   "this is a simple comment with a line number"

if the number of this line in the source file is 16. 

This basically answers the question [How to show line number when executing bash script](https://stackoverflow.com/questions/17804007/how-to-show-line-number-when-executing-bash-script) for ash users. 

Anything more to add? 

Sure. Why do you need this? How do you work with this? What can you do with this? Why do you want to tinker with this at all?

Debugging <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Lucky are those who have bash, but ash does not have LINENO, alas. Which is bad if you need to do a lot of testing on `Busybox` or the like.

That's why I finally developed a simple and powerful workaround with interesting side effects for debugging and logging purposes. 

Instead of using 

    echo $LINENO this is a simple comment with a line number

which does not work with e.g. `Busybox` or `Boot2Docker` or `Tiny Linux`, introduce a small and simple function `echo_line_no` (or whatever function name you like better) which prints the line number pretty much, but not exactly like LINENO. 

Line numbers are not just for fun, they are badly needed for debugging, and that's where things get more complicated. 

I could leave all of that as an exercise to you, but this is not the intention of Stack Overflow as I understand it. Hence I elaborate on the subject to give you as complete an insight as is possible for me. 

Sorry, it gets a bit lengthy for that reason. While I was developing my ideas, I digressed a bit into database specifics new to me. If you are not interested, please simply skip those sections.

In case you fear I get talkative or be prating, please stop reading. 

In particular I don't want to insult all you experts who are better than I, have far more experience and can do all of this easily on their own. 

It's for people like me I take the pain to write this all up, those who are new to the subject and have to fight their way more or less alone. Kind of paying back my debt according to the old mailing list ethics. In addition it turned out that this study revealed a lot of insight for myself as well.

Always bear in mind that I can only talk from my own experience, which is limited. So take this text with a grain of salt. Working on it, I added more and more of my daily routines and took notes of my investigation into realms new to me. That's far more than I initially planned for. And still I don't now what I'm heading for in the end.

Another caveat: As I took my notes while developing my ideas, I recorded all detours as well and didn't hesitate to document my stupidity. It is a question if all of these dumb attempts should have been cleared later. But even if you pursue the wrong way, you do learn a lot nevertheless. 

Contemplating about it, I decided to leave all that in place. It makes you look much smarter if you present a solution in a way that nobody can understand how you ever could reach it. This attitude is well known. But if you want to learn how to get to a solution, you have to study how this can be done from scratch.

Not those are smart who nod and pretend to understand, but those who dare to ask if they don't understand. That's the only chance to grow and become smarter. As a teacher, I prefer the latter type.

More complex cases <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Back to work. For example, if you like to use multi-line comments or show variables in your debugging messages, that simple approach given above will not work for those cases. 

Use the enhanced version instead: 

    echo_line_no () {
    #echo "--echo_line_no input :$1:--"
    # to see what is coming in when debugging
        input=$1
        NL='
    '
        input=${input%%"$NL"*}
        # for multi-line strings: cut off everything before $NL
        VARTOKEN=:
        input=${input%%$VARTOKEN*}
        # for variables: cut off everything before $VARTOKEN
    #    grep -n "$input" $0 | sed "s/echo_line_no//" 
    # can be done with grep alone, but cat adds spaces after line numbers, looks much nicer 
        cat -n $0 | grep "$input" | sed "s/echo_line_no//"
    } # echo_line_no

The result for the enhanced version using the test script `test_echo_line_no.sh` (see below) is:

    $ /path_to_your_script/test_echo_line_no.sh
    
    
    example 1:
        -- simple comment
        21   "this is a simple comment with a line number"
    
    example 2:
        -- without quotes (grep "$1") will filter for "ok" only, giving too many (multiple) results
        27   "ok for me"
        29   "ok for you"
    
    example 3:
        -- multiline comment
        35   "this is a multiline comment, will be cut off at new line
    
    example 4:
        -- variable substitution
        46   "this is another simple comment with line number and variable: FOO :$FOO:"
        >>>>>> : this is another simple comment with line number and variable: FOO :bar:
    
    example 5:
        -- variable substitution, 2 variables
        52   "this is another simple comment with line number and 2 variables: FOO :$FOO: BAZ :$BAZ:"
        >>>>>> : this is another simple comment with line number and 2 variables: FOO :bar: BAZ :42:
    
    example 6:
        -- show the value of a variable plus the line it is defined
        38  FOO=bar
        40  BAZ=42
        42  echo '
        15  MSG='a simple message'
        64  MSG='another simple message'
    
    example 7:
        -- simple call inside function without variable
        72  whatsup "hey"
    
    example 8:
        -- with variable and without VARTOKEN results in not showing at all
    
    example 9:
        -- with variable and with VARTOKEN
        86  whatsup "hi my dear buddy :$buddy:"
        >>>>>> : hi my dear buddy :joe:

How to use <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Create a script defining the function only. This script is to be included in the real script to be debugged:

    #!/bin/ash
    FILE=echo_line_no.sh  
    
    function echo_line_no () {
    #echo "--echo_line_no input -0- :$0: -1- :$1: -2- :$2:"
    # to see what is coming in
        input=$1
        NL='
    '
        input=${input%%"$NL"*}
        # for multiline strings cut off rest
        VARTOKEN=:
        input=${input%%$VARTOKEN*}
        # for variables, test only for stuff before $VARTOKEN 
       
        #    grep -n "$input" $0 | sed "s/echo_line_no//" | tee -a $log_echo_line_no
        # can be done with grep alone, but cat adds spaces after line numbers, looks much nicer 
        
        cat -n $0 | grep "$input" | sed "s/echo_line_no//" | tee -a $log_echo_line_no
        # if $log_echo_line_no is not defined, there is no error here
        
        # variable substitution
        case "$1" in
            *$VARTOKEN* ) echo "    >>>>>>> $VARTOKEN $1" | tee -a $log_echo_line_no ;;
            * )  ;;
        esac

    } # echo_line_no

Include this script into your working script (which you want to debug) via `source` call 

    source /path_to_your_script/echo_line_no.sh 

In consequence, you can use the function `echo_line_no` in your testing script anywhere you like, in particularly inside functions.

Here is the script used for testing the functionality of `echo_line_no` whose output was shown above:

    #!/bin/ash
    FILE=test_echo_line_no.sh   
    
    source /c/bak/echo_line_no.sh
    
    function whatsup {
    #echo "    debug ==:$1:====== argument given to function whatsup ========"
    # to see what we get
        echo_line_no "$1"
        # quotes are crucial -- otherwise 'hi my ...' 
        # would give all lines with "hi" like "this is...."  
    #    echo_line_no "this was from inside function whatsup, argument $1, line number is call line"
    } # whatsup 
    
    MSG='a simple message'
    
    echo '
    example 1:
        -- simple comment'
    
    echo_line_no "this is a simple comment with a line number" 
    
    echo '
    example 2:
        -- without quotes (grep "$1") will filter for "ok" only, giving too many (multiple) results'
    
    echo_line_no "ok for me"
    
    echo_line_no "ok for you"
    
    echo '
    example 3:
        -- multiline comment'
    
    echo_line_no "this is a multiline comment, will be cut off at new line
    second line of comment with a line number"
    
    FOO=bar
    
    BAZ=42
    
    echo '
    example 4:
        -- variable substitution'
    
    echo_line_no "this is another simple comment with line number and variable: FOO :$FOO:"
    
    echo '
    example 5:
        -- variable substitution, 2 variables'
    
    echo_line_no "this is another simple comment with line number and 2 variables: FOO :$FOO: BAZ :$BAZ:"
    
    echo '
    example 6:
        -- show the value of a variable plus the line it is defined'
    
    echo_line_no "$FOO"
    
    echo_line_no "$BAZ"
    
    echo_line_no "$MSG"
    
    MSG='another simple message'
    
    echo_line_no "$MSG"
    
    echo '
    example 7:
        -- simple call inside function without variable'  
    
    whatsup "hey"
    
    buddy=joe
    
    echo '
    example 8:
        -- with variable and without VARTOKEN results in not showing at all'
    
    whatsup "howdy $buddy"
    
    echo '
    example 9:
        -- with variable and with VARTOKEN'
    
    whatsup "hi my dear buddy :$buddy:"

Caveats <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

1. **Example 1**: Remember, all the magic stems from `grep`. Make sure each input string is significantly different to any other line in the script, as this is the token for `grep` -- otherwise you get more than that single line you want to see.

2. **Example 2** shows how important quotes are for the argument to the function -- try it without quotes to see the effect (`echo_line_no "$1"` vs. `echo_line_no $1`). Without quotes, only the first word is the trigger which will find 2 lines on each call here, so you get 4 results instead of 2, which will most likely be confusing.

3. **Example 3**: For multi-line strings, this constraint of uniqueness applies to the first line only as `grep` is line oriented -- the argument however has more than one line, so grep will fail and you see nothing unless we cut off everything after the first line. Consequently you will not see the other lines in the output, but that may not be really bad unless you need the information therein; if this is a problem, consider putting the information you need into the first line or add the token character for variables `VARTOKEN` (I prefer `:`) in each additional line you want to see.

4. **Examples 4 and 5**: For the use of variables, this same constraint of uniqueness applies to the part up to `VARTOKEN` used to enclose these (which is a good idea anyway to see if a variable is empty: `variable: FOO :$FOO:`). The reason is: `grep` looks for the original line and will not recognize the substitution (which we do not know), so the line has to be stripped from `VARTOKEN` as well. Instead, we show the variable values in the next line with the prefix `>>>>>> : `. 

5. **Example 6**: If you only give a quoted variable as argument (`echo_line_no "$FOO"`), you will not get the line number of the "comment" but the line number of the `definition of the variable` instead -- which may be exactly what you want as this information is hard to find otherwise. The first two show said variables `FOO` and `BAZ` from examples 4 and 5, the next 2 assignments show the lines of definition of the same variable `MSG` defined at different places with different values. 

    You may have noticed that we also got an enigmatic output here:
    
            40  BAZ=42
            42  echo '
    
    What does the last line mean? Where does it come from? Is it a bug? Or is it a feature?
    
    Well, it is neither a bug nor a feature but sheer luck that we see this phenomenon. 
    
    I could've chosen any value for the variable `BAZ`, but I picked [42](https://en.wikipedia.org/wiki/Phrases_from_The_Hitchhiker%27s_Guide_to_the_Galaxy#Answer_to_the_Ultimate_Question_of_Life,_the_Universe,_and_Everything_(42)) for no particular reason, and this is a line number also present in our file having some content, so `grep` cannot avoid to filter this line as well and show it, and there is nothing I as a programmer can do about it.
    
    Or can I? Of course I can. This number is the beginning of the line, and I could use that property as a token to filter out that result. I'm too lazy to do that and I don't see that this will really improve our debugging mechanism. The chances that this phenomenon will annoy under real circumstances are significantly low.
    
    What's really important here is that this phenomenon is well understood. That's enough.

6. You can use `echo_line_no` from inside a function. In contrast to bash there is no way to get the function name via system variable due to the same restrictions. No problem, you can always hardcode if necessary, as demonstrated here in the comment `this was from inside function whatsup`; you have to hardcode the call to `echo_line_no` there anyway. But there is more to the use within functions:

   6.1. **Example 7**: If the function argument does not contain a variable, the function shows the line number of the function call, like with variables. That's kind of a trace function, you see which line has called your function -- again something most valuable and hard to get otherwise. The line of the call within the function would be useless anyway.

   6.2. **Example 8**: If it **does** contain a variable and this variable is **not** masked by `VARTOKEN`, the whole line is not shown at all, as `grep` must fail due to variable substitution; in order to compensate this, call `echo_line_no` also for the variable itself to get the line number, as demonstrated. This behavior may be a bug or a feature.

   6.3. **Example 9**: If you do use `VARTOKEN`, the line number is shown as in the 3rd function example; this example also shows that inside the function quotes are crucial as well due to the same reason (`echo_line_no "$1"`), but in special cases you may even be interested in other places your first word appears (try it without quotes to see the result). Also, if a comment repeats the trigger, it will be shown, too, that's why the comment inside this function has been crafted carefully to not fall into this trap. You will notice anyway and know what to do, if it happens by chance. 

7. If you are like me and like to use `-` or `--` or `---------` as part of your parameter, you will have a problem, as grep will interpret this `-` as token for a parameter for grep and will fail and complain. Take something else.

Logging <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

You may even write a protocol for later inspection or tracking the performance of the working script. You simply define a variable `log_echo_line_no` in your script which holds the name of the log file 

    log_echo_line_no=/tmp/test_echo_line_no.log 

before calling `source /path_to_your_script/echo_line_no.sh`. That's all.

If you also want to clear the log file when starting the test script, simply add 

    echo > $log_echo_line_no

before the first call to `echo_line_no`.

Example <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

It may not be obvious why logging should be important in the first place. After all, while debugging, you will see the output on screen already, so you don't want to switch to the log file.

It's best to learn by example, so I give you a non trivial example of logging permanently. This example is interesting in another aspect. We can discuss if and when it is advisable to utilize prebuilt solutions to your problems.

Let's assume you have a fairly complex setup with a database engine of type MySQL or MariaDB which is accompanied by a replication setup of 2 slaves and a cron job polling your database replication slaves regularly for health status. 

    $ crontab -l
    # check mysql replication every minute
    * * * * * /path_to_your_script/mysql_repl_monitor.sh

(See Giuseppe Maxia: [Refactored again: poor man's MySQL replicator monitor](http://datacharmer.blogspot.de/2011/04/refactored-again-poor-mans-mysql.html). 

The resulting output of this script is logged to `/tmp/repl_monitor.log`, so you may permanently watch the health status by calling

    tail -f -n 60 /tmp/repl_monitor.log 

An example output with those 2 replication slaves might look like

    240 /path_to_your_script/mysql_repl_monitor.sh ------------------------------- 2018-02-27_12:23:00 
    175 =====> s1 2018-02-27_12:23:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82352145
    175 =====> s2 2018-02-27_12:23:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82352145
    
    240 /path_to_your_script/mysql_repl_monitor.sh ------------------------------- 2018-02-27_12:24:00 
    175 =====> s1 2018-02-27_12:24:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82424522
    175 =====> s2 2018-02-27_12:24:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82424522
    
    240 /path_to_your_script/mysql_repl_monitor.sh ------------------------------- 2018-02-27_12:25:00 
    175 =====> s1 2018-02-27_12:25:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82496899
    175 =====> s2 2018-02-27_12:25:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82496899

This is how it should be, regular entries at the given interval, nice and boring. 

If not, you see at a glance that something is wrong. Plus you see on which line of the script the output is generated and hopefully with the full error code from the replication instance reporting the error so you immediately know exactly what has happened where to which replication engine so you might take the right action based on this information in order to get replication to work flawlessly again. 

Line numbers aren't important if things are going smoothly, but if you are creating and debugging complex scripts like this one or -- more importantly -- if indeed an error occurs, you will be glad to know on the spot where to look.

So this should illustrate the importance of logging. Some final thoughts about the background of all this debugging business. 

Why roll your own? <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Why take the pain and write your own complex shell scripts including all that tedious debugging hassle? 

Can't you just utilize some pre-tested package provided by some great guy who knows how to do it and may even give it away for free? 

Well, yes, maybe, but rather: no. Let me explain.

The program of Giuseppe Maxia, the data charmer referred to above, sends an e-mail to alert a database administrator who has to take action in case something went wrong. In order to find out about this feature you have to read that script, understand it and draw your conclusions. What does this script try to do and what does it deliver? Is it what you expect and need?

So this insight doesn't come for free in the first place.

Next I may remind you that we put all these mechanisms in place because things do get wrong eventually.

Do you really want to rely on e-mail and wait for somebody to get ready and do something? Wouldn't it be better to automatically and immediately take action without human intervention? 

Of course it would be, and shell scripts are the perfect tool to accomplish this. The same argument obviously applies to watching the log file.

Giuseppe Maxia knows all that just as well, but it is not his job to do my job. He could not even do it if he wanted to, because he doesn't know anything about the nature of my setup. Therefore his intention is not to deliver a ready-to-use solution, but instead to outline how this task might be done in order to get you or me on the right track. So this is the first reason why we have to write our own scripts. But there is more to it.

Database replication error handling <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

We would need at least one more shell script to clean up the problem as soon as possible, and that is the moment an error is detected, which only depends on the interval of the monitoring service, itself being realized as a shell script. 

Giuseppe Maxia does not have a proposal of how to handle this problem. The classical way would be to inspect the nature of the problem first. Why did replication stop?

This is a good approach if you don't know what may happen. Maybe there is a logical or technical flaw in the application code causing this trouble. In this case it doesn't make sense to reset the slave and start again, because this error will undoubtedly appear again and again. You have to inspect your application code and make sure that no database error is induced by intention or carelessness.

But if you are sure that this doesn't happen and the error is not triggered and cannot be tackled by code, it may be a viable solution to apply brute force. If I understand the approach of database specialist company Percona correctly, this is what they do. They check at the level of the operation system if the corresponding tables are in sync, and if not, they copy the original master table to the slave. 

Here I don't write about fancy scenarios. I have experienced replication errors which obviously stem from the database engines involved and which I could not explain. Google of course knows about these errors. They are discussed in the MySQL forum, but none of these cases has found a solution. 

    Last_SQL_Error: Could not execute Update_rows_v1 event on table dj5.l_h_p; Can't find record in 'l_h_p', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000001, end_log_pos 571756

So there is nothing I can conclude here. Brute force is the only remedy.

Digression: Repair or copy <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

As these are the only errors I ever experienced apart from those induced by faulty code, I decided to develop a solution along the strategy of Percona. Thinking about it, it is an elegant solution as well. 

Consider for example the case that both master and slave tables differ because the slave table is corrupt for some reason (this may happen and usually nobody knows why -- could be hardware failure). 

There are 2 strategies here. Either you let the database engine of the slave repair the slave table (which you have to do anyway even if this error is not detected by sync checking). This still doesn't give you the guarantee that both tables are identical byte-wise. Or you copy the master table to the slave right away.

This (repair or copy) may take a lot of time depending on the size of your tables. For this reason, you may contemplate to partition big tables into sizable chunks. 

Chances are, only one file of the partition tables has a problem. In this case you only copy the faulty file which is most probably the fastest operation you can get.

Digression: Table partitioning <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

For example, I have more than a dozen MyISAM tables of the same type of data some of which with more than 1 GB datafile each. This will slow down rsync operations on those tables. 

Investigating this problem, I ended up dividing those tables into 3 groups, those small enough to not be partitioned, those to be partitioned into 100 chunks and into 1000 chunks. This way, I hoped to be able to keep table size in the range of at most 10 Mbytes. 

All those tables have an auto increment primary key id, so the partition formula is a kind of modulus operation given by

    ALTER TABLE tbl PARTITION BY hash (id * 100 + id) PARTITIONS 100;

or 

    ALTER TABLE tbl PARTITION BY hash (id * 1000 + id) PARTITIONS 1000;

respectively.

Looking at the biggest of those tables, only 605 of these 1000 partitions contain data so far, so the data obviously is not distributed evenly. The positive feature is that all data belonging to a particular id will be found within a single partition. 

As most queries ask for data correlating to a particular id, this means that only one partition table has to be opened. This justifies partitioning in the first place. Also, quite regularly deletes on these id are performed, and the performance hit for this operation is not as big if the table is partitioned.

With a table of different type, but the same property of giving exact hits in case of partitions, all 1000 partitions were filled quite evenly, so the principle as such seems to be correct and viable.

The average size of the partition tables is about 2 MB. The biggest chunk, however, has about 416 MB, which isn't quite what I was heading for. The situation doesn't seem to be that bad, though. The chance to hit one of the bigger partitions actually is much lower than hitting the whole unpartitioned table.

You may wonder about the peculiar formulas for the partition definition. They are due to the fact that the integer column used for hash partitioning starts with 1, which would make all records belonging to the first 100 or 1000 id would populate the first partition table. This formula avoids that. All of these first id records now belong to different partitions.

Digression: Partition by md5 <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Investigating the bigger tables in my collection, I noticed a kind of key-value-store based on a primary key given by a md5 value instead of an integer, as in the other cases. There is no algorithm for partitioning based on md5 values. 

Right now there is no reason yet to partition this table, which has only a couple of hundred rows but already nearly 100 MB of data, but in case it would make sense, the question is how to handle this case. Is it possible at all? And if so, how to do it best? 

Most of the articles I read discourage partitioning by `HASH`. The official MariaDB features [Rick's RoTs -- Rules of Thumb for MySQL](http://mysql.rjweb.org/doc.php/ricksrots) at [Partition Maintenance](https://mariadb.com/kb/en/library/partition-maintenance/). Rick James claims

    PARTITION BY RANGE is the only useful method. 

and doesn't even mention partitioning by `HASH`. 

There are lots of examples for partitioning by `RANGE`, but this is not really our business. We do have some tables where we collect data by date and will profit from this construction, but that isn't the main theme. So it may be interesting for others to see one more example for partitioning by `HASH` here. 

In order to get some idea, I first copied this table to a test database:

    >CREATE TABLE bak.tbl_md5 LIKE main.tbl_md5; 
    >INSERT IGNORE INTO bak.tbl_md5 SELECT * FROM main.tbl_md5;

The first idea was to transform the md5 value to an integer and then proceed as usual:

    >ALTER TABLE tbl_md5 PARTITION BY hash (CONV(md5, 16, 10) * 100 + CONV(md5, 16, 10)) PARTITIONS 100;
    ERROR 1564 (HY000): This partition function is not allowed

Therefore I introduced a new bigint column:

    >ALTER TABLE `tbl_md5` ADD `id` bigint unsigned NOT NULL AFTER `lg`;

Next I populated this column with the integer value of the md5 column:

    >UPDATE tbl_md5 SET id = CONV(md5, 16, 10);

Does it work now? No, it doesn't.

Digression: PRIMARY KEY clause <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

    >ALTER TABLE bak.tbl_md5 PARTITION BY hash (id_ct * 100 + id_ct) PARTITIONS 100;
    ERROR 1503 (HY000): A PRIMARY KEY must include all columns in the table's partitioning function

Oh yes, of course. 

When I encountered this error the first time it took quite a while for me to understand. Google was my first reach for help as usual, but everybody said: the error message tells it all. This didn't help me much.

With a primary key as the only unique key, things may be easier to understand, but I happened to tackle a table with an additional unique key and had to comprehend that **every unique key** has to be changed accordingly. 

The fix is easily done in any case:

    >ALTER TABLE `tbl_md5` ADD PRIMARY KEY `md5_lg_id_ct` (`md5`, `lg`, `id_ct`), DROP INDEX `PRIMARY`;

As you see, the primary key was a compound key to begin with, adding a language code to the md5 value. 

But now it should work, right?

    >ALTER TABLE bak.tbl_md5 PARTITION BY hash (id_ct * 100 + id_ct) PARTITIONS 100;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(`bak`.`#sql-180b_fc18`.`id_ct` * 100)'

How come? What is the biggest bigint value I can get from a md5 column? 

    >SELECT CONV('ffffffffffffffffffffffffffffffff', 16, 10);
    +--------------------------------------------------+
    | CONV('ffffffffffffffffffffffffffffffff', 16, 10) |
    +--------------------------------------------------+
    | 18446744073709551615                             |
    +--------------------------------------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('ffffffffffffffffffffffffffffffff', 16, 10) *100;
    +-------------------------------------------------------+
    | CONV('ffffffffffffffffffffffffffffffff', 16, 10) *100 |
    +-------------------------------------------------------+
    |                                 1.8446744073709552e21 |
    +-------------------------------------------------------+
    1 row in set (0.00 sec)

Oh, I see, the engine had to switch to exponential representation. Okay, the factor of 100 is not needed here and only makes things more complicated.

    >ALTER TABLE bak.tbl_md5 PARTITION BY hash (id_ct) PARTITIONS 100;
    Query OK, 361 rows affected (1.19 sec)
    Records: 361  Duplicates: 0  Warnings: 0

Finally it works. 

Digression: Unusable partition distribution <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

What is the result? Big surprise. 

I expected all the values to be distributed evenly over all partitions. But on the contrary all of them are in one partition. How come?

Well, which values may I expect?

    >SELECT CONV('00000000000000000000000000000000', 16, 10);
    +--------------------------------------------------+
    | CONV('00000000000000000000000000000000', 16, 10) |
    +--------------------------------------------------+
    | 0                                                |
    +--------------------------------------------------+
    1 row in set (0.00 sec)

No surprise here. We should expect the whole spectrum the other end of which we already saw, but obviously the distribution is by no means evenly. In fact we only have one single value for all the different md5 values. 

    >SELECT MIN(id), MAX(id) FROM bak.tbl_md5;
    +----------------------+----------------------+
    | min(id)              | max(id)              |
    +----------------------+----------------------+
    | 18446744073709551615 | 18446744073709551615 |
    +----------------------+----------------------+
    1 row in set (0.00 sec)

This is funny. It shouldn't be. In my understanding a hex value should be a number. Let's start simple.

    >SELECT CONV('f', 16, 10);
    +-------------------+
    | CONV('f', 16, 10) |
    +-------------------+
    | 15                |
    +-------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('e', 16, 10);
    +-------------------+
    | CONV('e', 16, 10) |
    +-------------------+
    | 14                |
    +-------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('c0', 16, 10);
    +--------------------+
    | CONV('c0', 16, 10) |
    +--------------------+
    | 192                |
    +--------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('cece', 16, 10);
    +----------------------+
    | CONV('cece', 16, 10) |
    +----------------------+
    | 52942                |
    +----------------------+
    1 row in set (0.00 sec)

Now this looks like it should be. Obviously this function `CONV` fails with 32-byte values:
    
    >SELECT CONV('3e5fb34e6dbf83ad19236125ffcece', 16, 10);
    +------------------------------------------------+
    | CONV('3e5fb34e6dbf83ad19236125ffcece', 16, 10) |
    +------------------------------------------------+
    | 18446744073709551615                           |
    +------------------------------------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('26b67a08ec79b047f6c21bcbad7109c', 16, 10);
    +-------------------------------------------------+
    | CONV('26b67a08ec79b047f6c21bcbad7109c', 16, 10) |
    +-------------------------------------------------+
    | 18446744073709551615                            |
    +-------------------------------------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('32e6588e792ae357eb4a05175cab87e', 16, 10);
    +-------------------------------------------------+
    | CONV('32e6588e792ae357eb4a05175cab87e', 16, 10) |
    +-------------------------------------------------+
    | 18446744073709551615                            |
    +-------------------------------------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('3d1ff61f4c797e2d63087715ff016f5', 16, 10);
    +-------------------------------------------------+
    | CONV('3d1ff61f4c797e2d63087715ff016f5', 16, 10) |
    +-------------------------------------------------+
    | 18446744073709551615                            |
    +-------------------------------------------------+
    1 row in set (0.00 sec)
    
This is what our conversion function delivers -- ever the same number. 

Digression: Experimenting with CONV <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Let's experiment with chopping off a part of this 32 byte md5 value.

    >SELECT CONV('236125ffcece', 16, 10);
    +------------------------------+
    | CONV('236125ffcece', 16, 10) |
    +------------------------------+
    | 38900156321486               |
    +------------------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('21bcbad7109c', 16, 10);
    +------------------------------+
    | CONV('21bcbad7109c', 16, 10) |
    +------------------------------+
    | 37094472224924               |
    +------------------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('a05175cab87e', 16, 10);
    +------------------------------+
    | CONV('a05175cab87e', 16, 10) |
    +------------------------------+
    | 176271729014910              |
    +------------------------------+
    1 row in set (0.00 sec)
    
    >SELECT CONV('87715ff016f5', 16, 10);
    +------------------------------+
    | CONV('87715ff016f5', 16, 10) |
    +------------------------------+
    | 148921010624245              |
    +------------------------------+
    1 row in set (0.00 sec)
    
It looks like I am on the right track.

    >UPDATE bak.tbl_md5 SET id = CONV(right(md5,10), 16, 10);
    Query OK, 361 rows affected (0.18 sec)
    Rows matched: 361  Changed: 361  Warnings: 0
    
    >SELECT MIN(id), MAX(id) FROM bak.tbl_md5;
    +------------+---------------+
    | MIN(id)    | MAX(id)       |
    +------------+---------------+
    | 1249212789 | 1097729061761 |
    +------------+---------------+
    1 row in set (0.00 sec)
    
    >UPDATE bak.tbl_md5 SET id = CONV(right(md5,5), 16, 10);
    Query OK, 361 rows affected (0.14 sec)
    Rows matched: 361  Changed: 361  Warnings: 0
    
    >SELECT MIN(id), MAX(id) FROM bak.tbl_md5;
    +---------+---------+
    | MIN(id) | MAX(id) |
    +---------+---------+
    |     622 | 1044978 |
    +---------+---------+
    1 row in set (0.01 sec)
    
    >UPDATE bak.tbl_md5 SET id = CONV(right(md5,15), 16, 10);
    Query OK, 361 rows affected (0.13 sec)
    Rows matched: 361  Changed: 361  Warnings: 0
    
    >SELECT MIN(id), MAX(id) FROM bak.tbl_md5;
    +-----------------+---------------------+
    | MIN(id)         | MAX(id)             |
    +-----------------+---------------------+
    | 585822353884923 | 1149761746146186111 |
    +-----------------+---------------------+
    1 row in set (0.00 sec)

This looks promising.
    
    >ALTER TABLE bak.tbl_md5 PARTITION BY hash (id) PARTITIONS 100;
    Query OK, 361 rows affected (0.48 sec)
    Records: 361  Duplicates: 0  Warnings: 0
    
    >ALTER TABLE bak.tbl_md5 REMOVE PARTITIONING;
    Query OK, 361 rows affected (0.69 sec)
    Records: 361  Duplicates: 0  Warnings: 0
    
    >UPDATE bak.tbl_md5 SET id = CONV(right(md5,5), 16, 10);
    Query OK, 361 rows affected (0.12 sec)
    Rows matched: 361  Changed: 361  Warnings: 0
    
    >ALTER TABLE bak.tbl_md5 PARTITION BY hash (id) PARTITIONS 100;
    Query OK, 361 rows affected (0.36 sec)
    Records: 361  Duplicates: 0  Warnings: 0
    
    >ALTER TABLE bak.tbl_md5 REMOVE PARTITIONING;
    Query OK, 361 rows affected (0.57 sec)
    Records: 361  Duplicates: 0  Warnings: 0
    
    >ALTER TABLE bak.tbl_md5 PARTITION BY hash (id*100 + id) PARTITIONS 100;
    Query OK, 361 rows affected (0.71 sec)
    Records: 361  Duplicates: 0  Warnings: 0

All of these experiments resulted in partition sizes which looked pretty similar: 2 or 3 empty partition tables, the rest filled quite satisfactorily. At first sight, there was not much difference. So truncating the md5 value by some sensible number should be the cure here.
 
At least I have found a solution to the md5 partitioning problem. And that's good.

Digression: Max value of bigint datatype <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

To see where things get wrong, I issued the following (concatenating 2 simple SQL commands to make testing easier -- sorry, reading is worse, but it may be interesting to see that this technique, which is well known from the shell, works in SQL as well):

    >UPDATE bak.tbl_md5 SET id = CONV(right(md5,15), 16, 10); SELECT MIN(id), MAX(id) FROM bak.tbl_md5;
    Query OK, 361 rows affected (0.21 sec)
    Rows matched: 361  Changed: 361  Warnings: 0
    
    +-----------------+---------------------+
    | MIN(id)         | MAX(id)             |
    +-----------------+---------------------+
    | 585822353884923 | 1149761746146186111 |
    +-----------------+---------------------+
    1 row in set (0.01 sec)
    
    >UPDATE bak.tbl_md5 SET id = CONV(right(md5,20), 16, 10); SELECT MIN(id), MAX(id) FROM bak.tbl_md5;
    Query OK, 361 rows affected (0.76 sec)
    Rows matched: 361  Changed: 361  Warnings: 0
    
    +----------------------+----------------------+
    | MIN(id)              | MAX(id)              |
    +----------------------+----------------------+
    | 18446744073709551615 | 18446744073709551615 |
    +----------------------+----------------------+
    1 row in set (0.00 sec)
    
    >UPDATE bak.tbl_md5 SET id = CONV(right(md5,25), 16, 10); SELECT MIN(id), MAX(id) FROM bak.tbl_md5;
    Query OK, 0 rows affected (0.05 sec)
    Rows matched: 361  Changed: 0  Warnings: 0
    
    +----------------------+----------------------+
    | MIN(id)              | MAX(id)              |
    +----------------------+----------------------+
    | 18446744073709551615 | 18446744073709551615 |
    +----------------------+----------------------+
    1 row in set (0.00 sec)

Maybe it is time now to write a bug report. 

Better not. 

The magical number 18446744073709551615 is 2^64-1 and the maximum of an unsigned big int. That's why! 

I never hit that number before. 

Digression: Table type: MyISAM vs. InnoDB <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

The data collected so far is just the beginning. Eventually all the partitions will be filled up quite evenly. Also, the overall size will grow accordingly, so we might have to introduce partitions by 10,000 - well, this is not possible at the moment, the limit is 8192, which, for a human only, makes it more difficult to compute the partition a particular id is to be found.

There are good reasons why I chose MyISAM as database engine for these tables and not InnoDB. The main reason of course is that we don't need any of the features discriminating InnoDB from MyISAM, not now in test mode and not later under heavy production conditions. 

With partitioned InnoDB tables, things are more complicated, as usual, but moving files around can be done nevertheless. See [Importing InnoDB Partitions in MySQL 5.6 and MariaDB 10.0/10.1](http://www.geoffmontee.com/importing-innodb-partitions-in-mysql-5-6-and-mariadb-10-010-1/)

In case of MyISAM you can even choose which method to apply if things go wrong, `repair` or `copy`. The `frm` file is not touched as a rule, so there is nothing to do. If only the data file `MYD` is different between master and slave, you best copy. `REPAIR TABLE` must copy as well, so this doesn't cost you any more time nor does it give you the guarantee that the data files are identical. 

If only the index file `MYI` is different, you best use `REPAIR TABLE`. Chances are, the repair is immediate, because very often it is simply the number of records which is wrong being zero or any other number different from the correct number.

Of course, if you copy, you have to make sure that the master table is not changed during these procedures. And if the process succeeded, you restart the slave reporting that problem and check if everything is okay now. 

End of digression.

Database replication regular health checking <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

One more consideration here. The monitoring script can only detect errors which occur during operation. If at startup the table setup on the slave is different from the master for some reason or the slave database engine produces an error maybe due to some hardware failure, the slave will never find out and the monitoring script is of no use in this case. Comparing the files directly however will find out and the copy process will successfully synchronize any files that are not in sync.

To show you how this could be utilized by way of precaution in a running [Docker](https://de.wikipedia.org/wiki/Docker_(Software)) system, later on I show a little script `mysql_rsync_lock.sh` which takes next to no time (in test mode) reading and writing to spinning disks if everything is okay (1-2 secs for 64 tables, about 5 GB disk space, 2 slaves), so it even might be run regularly. (As we'll see later, this is a bad idea in general because surprisingly data files may be different for no obvious reason while the table data is identical.) 

Well, this is not based on actual production data. In case your data moves quickly and indices are changed considerably in the process, the command `FLUSH TABLES` alone may take minutes. In production mode, we will use SSD and much more RAM, so those processes will be much faster. As far as we can see now, the whole database can be held in RAM.

You see, a solution to problems depends heavily on the nature of your setup.  If your system resembles Wikipedia, you have totally different problems from one that works like Facebook, eBay, YouTube or Google.

If you have more slaves, maybe even a large number, I would think of a kind of daisychain test. Compare the master with one of those slaves first, locking the master and this slave, then unlock both the master and the slave, give the slave time to catch up, then lock it again and the next slave to compare this one with the first slave, and so on. 

This way you will detect problems if replication traffic is not heavy, which means you don't have many time-consuming write processes which produce replication lag. I don't know if this will work. Data may be moving too fast for this kind of procedure. 

The benefit of this scenario obviously is that the master has to be locked for the comparison with one other machine only, no matter how many slaves you have. The slaves run in read mode anyway, and the penalty for locking is just slightly stale data, if any. The master being locked may not be acceptable at all -- it all depends on your setup.

You should utilize global transaction identifiers (GTID) if things get complicated. If you do your copy job, you know the GTID of the source, can make sure the GTID of the source is greater or equal than the GTID of the target, and can synchronize the target to exactly that same GTID before starting that slave. 

This way you can be sure to not leave any transaction out and to not duplicate any transaction which would produce trouble as sure as hell. Again, this is just an idea. If you really have severe problems, call the experts from Percona. Or develop your own solution and become an expert yourself.

For the repair process it would be intelligent to analyze the error message from the slave. If that shows that just one table is affected, then only this table has to be processed. Doing so will speed up the whole thing tremendously, in particular if you have lots of tables.

Database replication automatic failover <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Having a solution for occasional errors on slave machines doesn't mean your done. What happens when the master has a problem and this problem is propagated to the slaves? Well, I'm afraid there is no automatic solution for that problem.

But if the master stops or a problem can be detected on the master and the slaves are not affected, one of those may be elevated to be a master for the other slaves. There are 2 questions here. Which of the slaves should be the new master? How to realize the failover, that is how to tell the other slaves which is their new master?

Here you may feel that this whole scenario isn't just something for individual solutions crafted with shell scripts. It is a generic problem not depending on the operation system or the nature of the database.

The problem is as general as load-balancing. You wouldn't want to write a load balancer yourself. That's why I included [haproxy](http://www.haproxy.org/) as a docker container and let this container do the load-balancing work for the 3 database containers.

Just as well I could have used [MaxScale](https://mariadb.com/resources/blog/mariadb-maxscale-22-introducing-failover-switchover-and-automatic-rejoin) for this purpose, and indeed I have experimented with it at times when it was not yet mature. It's time to switch, I guess, so I will have a second look soon, because in addition to load balancing, MaxScale has some more features, one of them being automatic failover. And this is something you definitely want to have if you can get it. Of course there are alternatives, too, like [Master High Availability Manager](https://www.percona.com/blog/2016/09/02/mha-quickstart-guide/).

Adding a stopwatch <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

As I wanted to prove my claim with respect to the time taken, I noticed my chance to show you the power and elegance of Docker and improve the original debugging mechanism by adding a kind of stopwatch. 

Remember, `echo_line_no` takes exactly one parameter. If we add a second parameter and this parameter is "DATE", then we take the time and show it.

The property of `echo_line_no` not showing anything if a variable is part of the first parameter, which looks like a design flaw, now turns into a feature. We use the variable to suppress the output and only show the datetime. 

In the snippet below you see the 2 calls to rsync plus the time but no line number.

    docker@boot2docker:/tmp$ /path_to_your_script/mysql_rsync_lock.sh
        =========================================== 2018-02-27_00:12:57
    41: " -------- FLUSH TABLES LOCK TABLES WRITE" DATE
    48: " -------- rsync datm dat1"
    sending incremental file list
    
    sent 294,577 bytes  received 12 bytes  589,178.00 bytes/sec
    total size is 4,125,301,375  speedup is 14,003.58
        =========================================== 2018-02-27_00:12:58
    sending incremental file list
    
    sent 294,577 bytes  received 12 bytes  589,178.00 bytes/sec
    total size is 4,125,301,375  speedup is 14,003.58
        =========================================== 2018-02-27_00:12:58
    59: " -------- SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1"
    66: " UNLOCK TABLES FLUSH TABLES"
        =========================================== 2018-02-27_00:12:59
    75: " -------- done" DATE
    ---------------------------------------------- time taken 2 seconds


To implement this, add the first snippet to the top of `echo_line_no.sh`, 

    TIMEDIFF=3600
    # to compensate UTC when called by cron
    
    function get_date {
        DATETAKEN=$(date --date="@$(($(date -u +%s) + $TIMEDIFF))" "+%Y-%m-%d_%H:%M:%S")
    } # get_date 

The second snippet has to be put inside the function `echo_line_no`.

    if [[ ! -z "$2" && "$2" == "DATE" ]] 
    then
        get_date
        echo "    =========================================== $DATETAKEN " | tee -a $log_echo_line_no
    fi

And while we are at it, please insert the following snippet as well which guards you from showing the whole source file:

    if [[ -z $input ]]
    then
        echo "!!!!!!!!!!!! no input :$input: parameter given :$1:" | tee -a $log_echo_line_no
        exit
    fi 

A line like this 

    echo_line_no ": log_echo_line_no :$log_echo_line_no:" 

will trigger this error (<a href="#caveats-table-of-content">Caveat No. 1</a>). Change to

    echo_line_no "something very unique: log_echo_line_no :$log_echo_line_no:" 

This is the `rsync` synchronizing script:

    #!/bin/sh
    #FILE=mysql_rsync_lock.sh
    # we do not need to take the service down
    
    source /path_to_your_script/echo_line_no.sh
    
    pwd=$(pwd)
    
    # new standard: master on d, slave1 on e, slave2 on f
    DISK_M=$1
    DISK_1=$2
    DISK_2=$3
    
    # old standard: all on one disk
    if [[ -z "$1" ]]
    then
      DISK_M=d
      DISK_1=d
      DISK_2=d
    fi
    
    /path_to_your_script/service_start.sh  > /dev/null 2>&1
    # Make sure the whole setup runs 
    
    datm="/$DISK_M/data/master"
    dat1="/$DISK_1/data/slave1"
    dat2="/$DISK_2/data/slave2"
    # data directories for master and slaves
    db=main
    # we only syncronize this database
    
    # Unix timestamp
    BEGIN=$(date +%s)

    # take the datetime here
    echo_line_no " -------- FLUSH TABLES LOCK TABLES WRITE" DATE
    for db_server in m1 s1 s2 
    do
        docker exec $db_server mysql -e "use $db; FLUSH TABLES; LOCK TABLES WRITE;"  
    done
    
    echo_line_no " -------- dat_master dat_slaves"
    for db_dir in $dat1 $dat2 
    do
        sudo rsync -av --delete $datm/$db/ $db_dir/$db/  
        # in case of slave error rsync will find something
        echo_line_no " ---$db_dir---- " DATE
        # take the datetime here, suppress line, show only datetime 
    done
    
    echo_line_no " -------- SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1"
    for db_slave in s1 s2 
    do
        docker exec $db_slave mysql -e "STOP SLAVE; SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; START SLAVE;" 
    done
    
    echo_line_no " UNLOCK TABLES FLUSH TABLES"
    for db_server in m1 s1 s2 
    do
        docker exec $db_server mysql -e "use $db; UNLOCK TABLES; FLUSH TABLES WRITE;"  
    done
    
    echo_line_no " -------- done" DATE
    
    END=$(date +%s)
    
    let USED=$END-$BEGIN
    
    echo "---------------------------------------------- time taken $USED seconds"

Here we use `docker` in combination with the `mysql` client like a function which is extremely elegant and very powerful: 

    docker exec $db_slave mysql -e "STOP SLAVE; SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; START SLAVE;"  

You can do quite complex database operations this way.

Notice that for MySQL, `This option is incompatible with GTID-based replication` ([Replication Slave Options and Variables](https://dev.mysql.com/doc/refman/5.6/en/replication-options-slave.html#sysvar_sql_slave_skip_counter)). This restriction with respect to `SQL_SLAVE_SKIP_COUNTER` does not apply to [MariaDB](https://mariadb.com/kb/en/library/set-global-sql_slave_skip_counter/).

Obviously, the slave is in sync with the master after this operation. So skipping the offending operation on the slave will get the slave running again. 

At least I hope so. Maybe there is some more work to be done. The master has been locked and his binary log stopped at a certain position. Maybe we need to also synchronize the slave with respect to this position. Under heavy load you will certainly find out. Even in test condition, you can produce that load. Well, I should investigate this question. 

The script `mysql_repl_monitor.sh` polls every minute, as you see from the replication log and `crontab -l`, which is as frequent as cron allows. The call is cheap on the respective database engines, so there is no performance problem to be expected.

Digression: Docker and mysqlbinlog <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Another example: We use docker to get information from the slave which relay log it is working on:

    $ docker exec s1 mysql -e 'SHOW SLAVE STATUS\G'
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: mysql
                      Master_User: replica
                      Master_Port: 3306
                    Connect_Retry: 30
                  Master_Log_File: mysql-bin.000001
              Read_Master_Log_Pos: 73291
                   Relay_Log_File: mysql-relay.000002
                    Relay_Log_Pos: 5403
            Relay_Master_Log_File: mysql-bin.000001
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB:
              Replicate_Ignore_DB: tmp,bak
               Replicate_Do_Table:
           Replicate_Ignore_Table:
          Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table: tmp.%,bak.%
                       Last_Errno: 0
                       Last_Error:
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 5537
                  Relay_Log_Space: 73451
                  Until_Condition: None
                   Until_Log_File:
                    Until_Log_Pos: 0
               Master_SSL_Allowed: No
               Master_SSL_CA_File:
               Master_SSL_CA_Path:
                  Master_SSL_Cert:
                Master_SSL_Cipher:
                   Master_SSL_Key:
            Seconds_Behind_Master: 107
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error:
                   Last_SQL_Errno: 0
                   Last_SQL_Error:
      Replicate_Ignore_Server_Ids:
                 Master_Server_Id: 7727678
                   Master_SSL_Crl:
               Master_SSL_Crlpath:
                       Using_Gtid: Slave_Pos
                      Gtid_IO_Pos: 1-7727678-13
          Replicate_Do_Domain_Ids:
      Replicate_Ignore_Domain_Ids:
                    Parallel_Mode: conservative

Or even more compact:

    $ docker exec s1 mysql -e 'SHOW SLAVE STATUS\G' | grep Relay_Log_File
                   Relay_Log_File: mysql-relay.000002

These binary files are where information may be found when things go wrong -- at least they say so.

With the utility program `mysqlbinlog` we can read and export the binary log files to a file which is human readable in parts; in parts only because SQL instructions which may contain sensible data are encrypted. Online you will find recommendations to inspect the binlog file. You may even save the result to a file for thorough inspection. 

The correct syntax for the docker instruction is for example

    docker exec s2 /bin/ash -c 'mysqlbinlog -r /tmp/s2.mysql-relay.000002.sql /var/lib/mysql/mysql-relay.000002'

In order to be able to inspect the protocol file from the host, you have to map your tmp directory accordingly. This you do in the docker-compose yml file for master and slaves in case you use docker compose, for example:<span id="volumes"></span>

    volumes:
      - /c/tmp:/tmp
      - /d/data/master:/var/lib/mysql
      - /d/data/sphinx:/var/lib/sphinx/data
      - /etc/ssh/:/etc/ssh/

As you see here, we also use the [Sphinx database search engine](http://sphinxsearch.com/).

Digression: Partitioning by RANGE <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

In my opinion, due to the encryption, the log file isn't really useful. If you want to see what your database engine really does, you better record every data changing operation in a separate table, much like [Adminer](https://www.adminer.org/) does with it's history (`...&sql=&history=all`). 

As we don't utilize MaxScale (yet), we had to implement a master/slave-switch in our application anyway to send all data changing operations to the master and the rest to the slaves. 

That's where the logging mechanism belongs:

    $this->_connection_type = 'db_master';
    $this->_sql_log_record($sql);

The SQL term is compressed to save space, so searching for or looking at specific queries requires uncompressing. You cannot see the queries in your conventional Adminer interface.

MaxScale can do all that for you via [MaxScale Read-write splitting](https://mariadb.com/kb/en/mariadb-enterprise/mariadb-maxscale-21-readwritesplit/) and [MaxScale Query Log All Filter](https://mariadb.com/kb/en/mariadb-enterprise/mariadb-maxscale-14/maxscale-query-log-all-filter/), but of course you have to spend some time to understand all the bells and whistles or rather parameters and options. 

Rolling your own, however, you know exactly what you do. If you record all your data changing queries, that table may fill up very quickly, so you might implement a mechanism to regularly discard data as well.

This is another example for a good use of partitioning a table, this time by `RANGE`. The benefit is, that dropping lots of records by range cost nothing, as it is done immediately by dropping that particular partition. 

This regular dropping of the oldest partition and creating a new partition instead can be realized via stored procedure (you will find examples via Google, e.g. [MySQL Stored Procedure for Table Partitioning](https://gist.github.com/CodMonk/4b89294bbb48eb1edb31)) or conventionally via crontab, shell script and docker. Take your pick. 

Digression: Reorganizing sql logging <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

While I was reorganizing my tmp.sql_log table, I dropped the idea of using `RANGE`. You can drop a table or a partition very fast, that's true, but you can truncate a table just as fast. That's even better. You don't have to create a new partition regularly. You just recycle the partitions you have.

You may have noticed that I have called the database `tmp` here explicitly. The reason is that I don't want to replicate stuff I deposit there:

    $ docker exec s1 mysql -e 'SHOW SLAVE STATUS\G' | grep Replicate_Ignore_DB:
              Replicate_Ignore_DB: tmp,bak

What's the current state of the table definition?

    M:7727678 [tmp]>SHOW CREATE TABLE tmp.sql_log_log\G
    *************************** 1. row ***************************
           Table: tmp.sql_log
    Create Table: CREATE TABLE `tmp.sql_log` (
      `id_sql` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
      `tmstmp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      `sql_compressed` longblob NOT NULL,
      PRIMARY KEY (`id_sql`)
    ) ENGINE=MyISAM AUTO_INCREMENT=3064769 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci PACK_KEYS=1 COMMENT='test counter'
    1 row in set, 3 warnings (19.26 sec)

Here you see that I defined the prompt of the MySQL client to show if I am on a master or slave machine; that is what the `M:` stands for. After that you see the ID of that server which is a random number generated at startup.

The `AUTO_INCREMENT` counter shows that this log file has been in use for about 3,000,000 data changing operations already. 

In replication, every machine must have a different ID. If you have several slave machines, you can see by that number which is which. Slaves, of course, are characterized by `S:` respectively.

Oh, 3 warnings here... Better see what it is:

    M:7727678 [tmp]>SHOW WARNINGS;
    +-------+------+-------------------------------------------------------------------+
    | Level | Code | Message                                                           |
    +-------+------+-------------------------------------------------------------------+
    | Error |  145 | Table './tmp/tmp.sql_log' is marked as crashed and should be repaired |
    | Error | 1194 | Table 'tmp.sql_log' is marked as crashed and should be repaired       |
    | Error | 1034 | 1 client is using or hasn't closed the table properly             |
    +-------+------+-------------------------------------------------------------------+
    3 rows in set (0.00 sec)

No choice of what to do:

    M:7727678 [tmp]>repair TABLE tmp.sql_log_log\G
    *************************** 1. row ***************************
       Table: tmp.sql_log_log
          Op: repair
    Msg_type: status
    Msg_text: OK
    1 row in set (49.02 sec)

Well, the table had about 1.2 GB data, so it took some time. Here I chose the delimiter `\G` to show you the difference. This feature is particularly interesting with extremely long rows. 

Everything okay now? Test it.

    M:7727678 [tmp]>SELECT id_sql, uncompress(sql_compressed) FROM tmp.sql_log_log ORDER BY 1 LIMIT 1\G
    *************************** 1. row ***************************
                        id_sql: 1
    uncompress(sql_compressed): DROP TABLE IF EXISTS tmp.tn_id_ar, tmp.tn_id_bn, tmp.tn_id_de, tmp.tn_id_en, tmp.tn_id_es, tmp.tn_id_fa, tmp.tn_id_fr, tmp.tn_id_hi, tmp.tn_id_it, tmp.tn_id_ja, tmp.tn_id_nl, tmp.tn_id_pt, tmp.tn_id_ru, tmp.tn_id_ur, tmp.tn_id_zh
    # L: 1926. F:/www/application/models/Ex_model.php. M: Ex_model::_drop_tmp_tn
    1 row in set (0.00 sec)

The code is pretty much self-explanatory. May I point you to the comment in this SQL statement?

Digression: Comments and editors <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

If you have a fairly complex application, you want to know where to look when an error occurs. That's why I made it a habit to add this kind of debug information to every single SQL query in my code. I want to see the line, the file and the method which has called this database query. 

    # L: 1926. F:/www/application/models/Ex_model.php. M: Ex_model::_drop_tmp_tn 

Maybe there are even more calls in between, so I get something like a trace. I have 2 mechanisms for this. The first is used when I don't want to clutter the SQL term with debug information. I then simply insert the line

        $this->dba->comment = "# L: ".__LINE__.". F: ".__FILE__.". M: ".__METHOD__;

right before the query. That's obviously PHP, and in order to understand this line, I have to tell you more.

I use [CodeIgniter](https://codeigniter.com/), and as far as I remember, in the earlier days they had a module named `Active_record`. Anyway, I wrote an `Active Record Class` which takes care of everything I like. This is what the `dba` stands for. So this class has a property `comment`.

    public $comment = '';

The query method of that class makes use of this property. That's all.

I don't write this line by typing, that would be cruel. Instead I use [AutoHotkey](https://autohotkey.com/) extensively, so I might have defined a hotkey to produce this line. 

AutoHotkey or short AHK works under Windows from everywhere, but, this being PHP code, I need this particular term in my favorite editor [PSPad](https://www.pspad.com/en/) only. And this editor has its own hotkeys or rather expansion of shortcuts into whatever you want (`AutoCorr.TXT`) -- pretty much like Word for Windows, plus some more functionality, which you can even extend via JavaScript, for example. Example for expansion:

    inss|$this->dba->insert($db_table, $ar_set, "L: ".__LINE__."\n#F: ".__FILE__."\n#M: ".__METHOD__);

Every once in a while I look for other editors in case the technical development has produced something more productive than PSPad. My latest adventure in this direction was [Atom](https://atom.io/), but it turned out to be of no use for several reasons.

One of them was that one property was not usable, so I contacted the person in charge to discuss things with him. He answered that he actually did have implemented it the way I wanted to, but the community had decided otherwise. And that was that. 

Sorry. I don't want to hack my own editor. I only want to be productive. And I'm happy and very productive with PSPad. By the way the shortcut for the comment line is `tdc`, for $**t**his->**d**ba->**c**omment.

Apart from that, I can always add the term

    "# L: ".__LINE__.". F: ".__FILE__.". M: ".__METHOD__;

to whatever code I have in place. Of course, many more complex ready-to-use SQL statements already have this line integrated, actually all of them. That's why although I do have a shortcut for this snippet, I simply don't need it.

Digression: Other tools <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

So for this shortcut expansion in PSPad I don't have to clutter the AHK namespace, which is crammed full anyway. To give you an example for AHK shortcut expansion I choose some from the database realm:

    ::saf::SELECT * FROM 
    ::sbl::SHOW BINARY LOGS;
    ::scf::SELECT COUNT(*) FROM 
    ::sct::SHOW CREATE TABLE \G 
    ::sdb::SHOW DATABASES;
    ::sss::SHOW SLAVE STATUS\G
    ::ssu::SELECT user, host, password FROM mysql.user ORDER BY 1; 
    ::ssv::SHOW VARIABLES LIKE 'serv%';   
    ::stt::SHOW TABLES;
    ::sttt::SHOW TABLES FROM tmp;
    ::svl::SHOW VARIABLES LIKE '%%';   
    ::sww::SHOW WARNINGS;

I use those extensively from [SuperPuTTY](https://github.com/jimradford/superputty) (after having tried numerous other SSH clients for too long a time with more or less trouble): 

SuperPuTTY is my window to my Linux workhorse on the same network. This box is booted from a stick with boot2docker. The docker containers I work with don't live in virtual machines like Vagrant on the same box, but rather more production-like on this separate machine. 

For example, this is the command which is executed automatically via PuTTY configuration and saved as layout to open a mysql session to my master engine `m1` and main database `dj5`: 

    docker@boot2docker:~$ /path_to_your_script/mysql_start.sh dj5

This script reads:

    #!/bin/sh
    #FILE=mysql_start.sh
    
    DB=$1
    
    if [[ -z $1 ]]
    then
      DB=tmp
    fi
    
    docker exec m1 mysql -e "SET GLOBAL max_binlog_stmt_cache_size = 2097152000;"  
    docker exec -it m1 mysql $DB 

Mind the parameter `-it` in the second call to `docker exec` here. It makes sure that you get a window (**i**nteractive **t**erminal) to work with.

Once in that mysql session, you might want to see the process list because some processes hang which indicates that there is another process with a lock on a table the other processes want to use and can not until that first process ends and releases this lock. You want to see what kind of long-running process that is in order to get things right, maybe by killing this process.

You may then utilize your keyboard and type bravely `SHOW PROCESSLIST;` -- until one of these days you say "I don't want to type this anymore" and define an AHK hotkey:

    ::spl::SHOW PROCESSLIST;

You see it costs me next to nothing to call `SHOW WARNINGS;` or `SHOW CREATE TABLE \G` like above. And whenever I feel the need for some more ease in my work, I use AHK to define something new. My latest addition is

    ::ell::echo_line_no ""

Here are some other snippets I use often:

    ::ggrr::grep -rn '/mnt/sda1/xq/dj/application/' -e '' ; search for terms in source code
    ::ggc::git clone
    ::ggcc::git checkout  
    ::ggs::git status
    ::ggp::git push origin master
    ::ggpp::git pull origin master    
    ::mms1::docker exec -it s1 mysql dj5                  ; start mysql session on slave 1
    ::mms2::docker exec -it s2 mysql dj5                  ; start mysql session on slave 2 

The best are more complex commands which really do good work. For example, I placed a command to the Windows key plus o (denoted in AHK lingo: `#o`) to immediately jump to the function definition in my file, when the cursor is placed on the function name. 

Likewise, `#k` will produce a list of all the function calls of that function. `#j` will produce a list with all the lines containing the word the cursor happens to be placed on. And so on. The limit is only your imagination. These are real productivity boosts; to achieve this by typing you would have to type a couple of keystrokes and combinations of those again and again. 

Although I use AHK for years now, there is hardly a day that I don't open the AHK editor. Okay, sometimes I have to look up what the right shortcut is as I forgot due to scarce usage. It's important to define shortcuts you can easily remember under all circumstances. But use it with caution, Windows may not behave like you want it to. I have spent numerous hours trying to get it right. I'll show you another example later on.

Another principally simple and nifty command I use regularly reads like this:

    git checkout temp$NO && git merge -s ours master && git checkout master && git merge temp$NO && git push origin master && git checkout temp$NO && git branch -a --color 

That's quite a long line and you don't want to type that in. What does it do? Well, usually I use the branch temp for git. If I get screwed up, I might define the shell variable `NO` and branch out to `temp$NO`, say `NO=2`. If that's okay and I feel like pushing to the master, I call the above sequence.

It checks out the branch I am in and merges this to master and checks out master, and -- yes, it merges the temp branch back -- and then pushes the master to origin and then checks back to the temp branch I was in and shows my branches in colors. Great. All that with just a few keystrokes you have to remember.

I'm sorry, I cannot explain why it merges the temp branch back -- I didn't construct this, I found it somewhere online and found it extremely useful. I didn't record the URL, though, which I do quite often, but not here, unfortunately. If you Google for `git merge -s ours master && git checkout master` you find a Russian page with a similar sequence, that's it. I don't speak Russian, so I didn't find it there. The original must've been lost, at least to Google.

Your creativity will find lots of situations where you can ease your workload. 

One more tip: for the task of recording clipboard snippets I also used a number of other tools, but none of them were really good in the long run. The Clipboard Manager [CopyQ](https://hluk.github.io/CopyQ/), however, is absolutely excellent. 

I have defined `F1` as the hotkey to the CopyQ clipboard list and defined a second tab in CopyQ which lists all images, to keep both parts apart. Also I have enlarged the available space as much as possible. I can afford this and don't want to lose anything I have copied for some reason. This tool is very fast and has a very efficient search engine. 

My workspace in [WinSCP](https://winscp.net/) is automatically opened and includes tabs with the 15 most frequent directories on my Linux machine. I use WinSCP in Explorer mode to launch any files for editing in PSPad. For file transfer I use [FileZilla](https://filezilla-project.org/).

Digression: Speech recognition <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

I can't resist. The sentence above "Great. All that with just a few keystrokes you have to remember." brought back to memory an incident of about 1995. Dragon's first product hit the market. I guess it was called something like 30k because it could remember 30,000 words. It worked on MS DOS, needed some piece of hardware, and was quite expensive, but for professionals like lawyers, who have to dictate lots of text every day, this investment might have made sense.

Back then we were serving this profession and we offered them this product. Of course, I had to prove that it worked, and the climax of the show was when I demonstrated the capability of defining complex macros triggered by just a few words. Those words could have been made up. The machine would learn this "word".

A lawyer, after having produced his text, usually by normal dictation, would give this to his secretary who would listen to this dictation and type the text to whatever machine she has at hand. Back then it was the time when many offices still had electric or electronic typewriters, no computer. Sometimes they had dictation machines which looked like being 50 years old or more. Why, they still worked fine.

The secretary would then have to save this legal document, print it for the client, the opposing party, the attorney, the court, maybe several consultants and witnesses. And maybe she will also use his new acquisition, the fax machine, which was the latest technical equipment of the time. 

With a computer and the dictation system, the lawyer would be able to produce the text file himself and also do the rest, that is producing all those copies, which isn't what he is trained for. The term I coined for this complex operation was "fax it off". And of course, it worked.

The argument was not that he didn't need a secretary anymore. He probably would dictate into his dictation machine faster and with more ease than into the new speech recognition engine. But he could have been productive even in non-office times, when there was no secretary around. And this did happen.

Digression: WSR vs. DragonDictate <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

I wouldn't write this long text when I had to type it. Instead I use DragonDictate. I work with Windows Speech Recognition (WSR) as well, using Windows for my mother tongue and DragonDictate for English, both in parallel. You cannot do this with DragonDictate, you have to dump one language and load the other and then back again, which is tedious.

Windows Speech Recognition works equally fine, although it uses other terms to navigate the speech engine. That's really bad, but, hey, we are human, that shouldn't be a problem for us. And it isn't. You can learn that, too, and switch the navigation terms. Both speech recognition programs interfere badly with some other programs. Most probably because those speech recognition programs are way too smart these days. 

DragonDictate, for instance, if you say one word only, because you're still thinking about the rest of the sentence, will search all tabs in your application for this word in order to switch to that tab. It took me a long time to understand what's happening here. It's really annoying if your machine all of a sudden does something which you didn't expect and cannot understand.

At the beginning of the new century, I taught database classes and sometimes used DragonDictate in class to dictate SQL into my notebook. Still, I don't program with DragonDictate (except for comments, which isn't really programming), instead I rather use AHK- or PSPad-defined shortcuts. But as soon as I have to write more than a few characters, I'll switch to DragonDictate or Windows Speech Recognition. That's one of the reasons why I would never be happy to use Linux as a desktop system.  

Well, I noticed I got tired of typing after having thoroughly investigated a very complicated problem for hours. I switched to dictating SQL, and it was really relieving. So nice. I should do that more often. I should rather make it a habit.

Digression: Dictation workflow <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Speech recognition programs don't work equally well with every other program. That's why they have a special simple text input program, a dictation pad.

I don't use those. I use [Flashnote](http://softvoile.com/flashnote/) instead for dictation input. This tool has a hotkey to be called to front, which I set to `F7`. So this is my workflow: I am in my document and see something to be improved. If it is a simple addition, I click `F7`. If I have to change something, I mark this, copy to the clipboard (my keyboard has a special key for this), switch with `F7` to Flashnote and insert the clipboard content, again with another special key of my keyboard.

Next I choose the appropriate speech recognition engine via `Pause` or `F9`. I do my work, and when I'm done, I may put the speech recognition engine to sleep or not, but in any case I transfer my work with `F10` to the application I was working on before. That's really convenient and I have produced huge amounts of text in both languages this way.

If in the meantime you have changed to another program, like I did right now to find the URL of Flashnote, you better first switch back to the program you want the text to be inserted. So the workflow now is `F7 F10`. `F7` to get back to Flashnote, `F10` to transfer the text. Which is what I will do right now.

Of course you can dictate in many programs directly, and I do that as well. The problem with Windows Speech Recognition is, that it starts always in caps mode except when it should start in caps mode, and I don't know how to flip that behavior. DragonDictate on the other hand always starts in small letter case, so that is easier to compensate by saying `Caps on` before dictating. Or just the other way around, depending on the program you are dictating to.

Also, chances are it is next to impossible to correct the speech engine in these programs; instead you get a long sequence of keystroke instructions or something else which screws things up. A better way to cope with correcting is to just mark part of the text and then dictate anew. That's a workaround for Windows Speech Recognition as well: start your marking with a word which is capitalized anyway.

A caveat here: with speech recognition, you will hardly make spelling errors, but you will undoubtedly get hearing defects. Not only you will get `there` and `their` mixed up, but also `marking` and `marketing`. And this time no automatic spelling correction can detect these errors. You really have to be attentive and proofread thoroughly.

Digression: AHK magic <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

My hotkey to turn DragonDictate on or off is the `Pause` key which usually is of no use and sits very prominently on the keyboard to not be missed easily. Windows Speech Recognition is toggled with `F9`; `F10` will copy all content in the open program to the one opened before. 

This `F9` `F10` is again AHK magic. 

    #IfWinNotActive, ahk_class wxWindowNR							; only when not in FlashNote
    F10::
    	Sleep 600
    	Send ^c
    	Sleep 400
    	SendInput, {Ctrl down}{Ctrl up}
    	SendInput, {Alt down}{Alt up}
    	Send !s
    	WinWaitActive, ahk_class wxWindowNR
    	IfWinNotActive, ahk_class wxWindowNR
    	{
    		MsgBox WinNotActive Flashnote
    		return
    	}
    	Sleep 550
    	ControlFocus, Edit2 ; 
    	Sleep 150
    	Send {Alt up}
    	Sleep 1150
    	Send ^v
    return

If there is a program who uses these keys, you can exclude them.

    #IfWinNotActive, SuperJPG - Registered to xxx
    F9::
    	toggle_wsr()
    return

    toggle_wsr() {
    	SetCapsLockState Off	
    	SendInput, {Ctrl down}
    	SendInput, {LWin down}
    	SendInput, {LWin up}
    	SendInput, {Ctrl up}
    }

This looks a little bit funny; I defined a function here because this function is used in other circumstances as well.

Digression: Key trouble <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

I thought I found a solution for a problem that bugged me for years. And the reason is that I described in detail my workflow and that I use the key `F9` to toggle Windows Speech Recognition. Quite frequently, I experienced PSPad to freeze. It took me quite a while to find out that, although I didn't want that, Windows Speech Recognition had been turned on, and as a consequence, PSPad totally froze. I had to kill both.

To find out about this coincidence I communicated with the creator of PSPad about that phenomenon, but he didn't have a clue. So finally I decided to no longer use Windows Speech Recognition and toggle DragonDictate instead. That's okay if I work for a long time in one of these languages, but if I have to switch very often, that's really no longer acceptable. And right now, working on this text, that's the case.

So I activated Windows Speech Recognition again and experienced PSPad freezing, which is when I remembered why I quit using it. Before turning back to the second-best solution, I had an idea. What if it is only the wrong key? Would I get the same phenomenon with another key? So I simply redefined AHK to `F8`. The problem is gone.

What's more, it turned out that I originally had chosen that key `F8` and used it in connection with `F4` to open Dragon DictationBox. Somehow I must have decided that the combination `F9 F10` is much more pleasant than the combination `F8 F10`, which introduced trouble. I'm glad I'm back to smooth operations. Hallelujah!

Well, cheered too soon! It turned out that the freezing is triggered by one of the AHK keys I have defined, for example `Win+j`

    #j::           ; find all and display as list
    Send ^f!l 
    return

Which looks innocent enough. It just says: `Ctrl+f`, then `Alt+l`. Let's see if I can reproduce it. Yes, I can. [ProcessExplorer](https://en.wikipedia.org/wiki/Process_Explorer) shows a CPU load of nearly 40% in one of 2 PSPad-Windows I have open right now. This should be the one in trouble. I am correct. Killing this one removes the frozen PSPad instance. It is a really big PHP file I'm debugging. The other is the markdown file with this text, which is fine.

So this is good. I have a test case. My experience tells me to test if these 2 keystroke combinations are sent too fast. So I changed the AHK code:

    #j::           ; find all and display as list
    Send ^f
    Send !l 
    return

That's almost perfect. The Windows Speech Recognition icon in the taskbar flashes, but PSPad doesn't freeze. I changed the code again:

    #j::           ; find all and display as list
    Send ^f
    Sleep 100
    Send !l 
    return

Big surprise! This time I get the Windows greeting image immediately and am logged out. What is this? `Win+L` does log me out, but `Alt+L` should not.

Increasing `Sleep` to 200 ms produces the error: I can see very clearly the result of the command `Ctrl+f`, and then the Windows Speech Recognition engine is turned on and I am logged out. So `Sleep` is not the solution.

You may have noticed that I used a very awkward procedure in the function `toggle_wsr` which was necessary due to very similar problems, if I remember correctly. So this is worth a try,

    #j::           ; find all and display as list
	SetCapsLockState Off	
	SendInput, {Ctrl down}
	SendInput, {f}
	SendInput, {Ctrl up}
	SendInput, {Alt down}
	SendInput, {l}
	SendInput, {Alt up}
    return

Delivers the same result, unfortunately. What to do now? Leave it almost perfect? No. This time PSPad froze again. I guess I just quit Windows Speech Recognition when I switch to work with PSPad. Windows Speech Recognition starts relatively fast, so I might have to live with that workaround for now. Or else I need to abstain from using those nifty shortcuts.

There is one more observation with respect to Windows Speech Recognition. When starting, some programs are heavily "touched" and show this by flickering and presenting the Windows system dialogue reading something like "server is busy -- switch to another application". 

When I had PSPad open and shut down Windows Speech Recognition, the code explorer window was flickering as well. So obviously Windows Speech Recognition interferes with other programs which should not be.

Digression: Explanation, no solution <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

This situation doesn't leave me at rest. The PHP file I was working on is very big. For testing, I'll better change to a small file. Starting Windows Speech Recognition doesn't work. ProcessExplorer reveals that `sapisvr.exe` is still running, so I kill it. Now the program starts. But while it is starting, Flashnote flickers and DragonDictate does not respond before the start process ends. 

That shouldn't be. A program shouldn't interfere with other programs this way. I concede that a speech recognition program must correspond with other programs, but only if it is put to work.

Well, it looks like I found an explanation, but no solution. The explanation reads like this [Windows Speech Recognition + AutoHotkey = Weirdness ](https://autohotkey.com/board/topic/90761-windows-speech-recognition-autohotkey-weirdness/):

>WSR seems to listen to any combination of Win and Control, triggering on the release of the second key:
>
>Win down, Control down, Control up
>
>or
>
>Control down, Win down, Win up

A very elegant solution would be to redefine the key combinations from `Win+?` to `Alt+?`, but this doesn't work in PSPad and some other programs; they seem to catch all `Alt` combinations to validate them or kill them otherwise. `AltGr` did not work either.

Choosing a much smaller PHP file shows that PSPad does not freeze on that one. But this doesn't help me either.

There are 3 solutions I found. 

1. PSPad has an own macro recorder. This would be the most elegant solution. Unfortunately, the find dialog cannot be macro recorded.                      

2. Another would be to write a JavaScript program for PSPad for each of my hotkeys, which would be overkill, I guess. 

3. The other is found in the above link: simply switch the `Win` key `#` with a key combination of `Shift+Ctrl` or `Ctrl+Shift`: `^+`. 

Problem solved. Finally.

Digression: Pronunciation <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

A nice feature of dictating is that it costs nothing to write complicated long words like `DragonDictate` or `Windows Speech Recognition`. I don't have this problem in my mother tongue, but when DragonDictate doesn't understand a word correctly I oftentimes suspect that my pronunciation is wrong.

In that case I switch to [dict.cc](https://www.dict.cc/), look up this word and listen to different speakers. It is amazing how English words can be pronounced as such. I still remember the embarrassing situation when I was in the USA in the late 60s and had to go to a hardware store to ask for a `gauge`. 

I had looked up this word in my printed dictionary and couldn't imagine of how to pronounce this word. The people in the store didn't understand what I wanted until I wrote this word down, whereat they were relieved and pronounced it the way it is pronounced. [Wiktionary](https://en.wiktionary.org/wiki/gauge#English) tells me this word comes from French. I could've figured out myself. 

But still many words are pronounced very differently by different speakers. Well, this isn't surprising, it is true for all languages there are, I guess. That's why the DragonDictate engine asks you if you want to choose the British or American version (I chose the latter one). If I pick up the pronunciation of an American speaker, as a rule it works perfectly. Too bad that the influence of my adored British schoolteacher creeps up again and again not to speak of my unavoidable foreign accent. 

And then there are the words I have no idea how the correct pronunciation might sound. If I didn't have the Internet, I wouldn't know what to do.

Long words are no problem, short words are. Recently I looked up the pronunciation for `their` and `there `. Surprisingly, all speakers pronounced those words absolutely identical. No chance for a machine. And no chance for me.

Digression: Hello computer <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

If I don't have speech recognition at my disposition, I feel like crippled. I use speech recognition on my notebook just the same when travelling. Traveling kills lots of time for nothing. 

When I was traveling a lot, I used to dictate into my notebook on the plane, on the train, on the bus, waiting in the lobby, in my hotel room, just about everywhere when I had some time to do some work. Nowadays nobody cares if somebody is talking to his smartphone, but even then people hardly took notice. Never anybody asked me what I was doing. 

I don't use the latest edition of DragonDictate as I don't see the need to buy this product again and again. It's excellent, at least for my purposes, and I don't even have the professional edition, so I can't use any macros (version preferred 10.10, must be 10 or rather 15 years old now).

Every once in a while, when I met colleagues complaining about stress injury syndrome, I told them about this fascinating technique. You cannot have it in Linux, unfortunately, except you simulate Windows, but anyway I still have to meet somebody who is interested. I don't know of anybody who dictates.

Of course, there are lots of people online using these products and chatting about it in their forums, but they are a totally different kind of people and professionally producing huge amounts of text. The industry has concentrated on lawyers and physicians, obviously successfully. Although back then everybody was dreaming of talking with a machine (see Star Trek IV: ["hello computer"](https://www.youtube.com/watch?v=v9kTVZiJ3Uc), 32 secs), not much has happened in the private realm. Nowadays we have Siri and Cortana, but sorry, I don't use that. 

Yesterday, somebody told me that she switched to dictating because of [WhatsApp](https://www.whatsapp.com/). In consequence, her messages get longer and longer. That's good.

There is much discussion about the right type of microphone to use with speech recognition. As long as I use speech recognition, which is more than 20 years now, I tried many different types most of which were pretty cheap, whereas all those professionals swear that you have to spend as much money on the microphone as you can afford. I cannot confirm that.

The last years I used a microphone which I cannot buy anymore, unfortunately. It has an ear hook and a solid short microphone arm, is easy to wear and works excellently. Hopefully it will not be damaged by accident in the years to come. It looks like I cannot replace it. I'm sure I didn't spend as much as $5 on that nice piece of technology.

Digression: Partitioning by day <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Back to partitioning. I don't need that old data anymore. So to save time testing partitioning, I `truncate` that table.

    M:7727678 [tmp]>TRUNCATE TABLE tmp.sql_log;
    Query OK, 0 rows affected (0.16 sec)

For partitioning the table by date, the first thing I had to change was the definition of the timestamp column. I cannot use the type `timestamp` for partitioning, but `datetime` is fine.

    ALTER TABLE `tmp.sql_log` 
    CHANGE `tmstmp` `tmstmp` datetime NOT NULL ON UPDATE CURRENT_TIMESTAMP AFTER `id_sql`;

Next I had to change my code because a `timestamp` value will be populated automatically, whereas a `datetime` value has to be set manually. Partitioning by day doesn't make sense when every timestamp has value `0000-00-00 00:00:00`. No big problem.

Thinking about it, I am wrong. Adminer set the wrong value by default. I just have to set the correct default value.

    ALTER TABLE `tmp.sql_log`
    CHANGE `tmstmp` `tmstmp` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP AFTER `id_sql`;

Also, I decided to add a new column to my logging table. Many queries target a special ID, and this ID should be part of the record as well to be able to pick all the queries belonging to this ID. Your mileage will vary. Nothing hinders you to add whatever you want.

    ALTER TABLE `tmp.sql_log` 
    ADD `id_ex` bigint(20) unsigned NOT NULL AFTER `id_sql`;

As I want to partition with respect to this`tmstmp` column, I have to make sure that this column is part of the primary key -- or more correctly -- of every unique key.

    ALTER TABLE `tmp.sql_log` 
    ADD PRIMARY KEY `id_sql_tmstmp` (`id_sql`, `tmstmp`), DROP INDEX `PRIMARY`;

Finally we are ready to partition this table. I think the 7 days of the week are a convenient roster to work with in this case. We wouldn't like to keep this data for a whole year or month.

    ALTER TABLE `tmp.sql_log` PARTITION BY HASH (DAY(tmstmp)) PARTITIONS 7;

As the day number of today is 28, the modulus by 7 of which is 0, partition 0 should be effected. To test it, I initiated an operation which would change data, got 154 entries in my table `tmp.sql_log` in partition `p0`. To make sure I'm on the right track, I issued a couple of entries manually to get different timestamp values:

    M:7727678 [tmp]>insert into tmp.sql_log (tmstmp) values ('2018-02-27 12:59:00');
    Query OK, 1 row affected, 2 warnings (0.01 sec)
    
    M:7727678 [tmp]>SHOW WARNINGS;
    +---------+------+-----------------------------------------------------+
    | Level   | Code | Message                                             |
    +---------+------+-----------------------------------------------------+
    | Warning | 1364 | Field 'id_ex' doesn't have a default value          |
    | Warning | 1364 | Field 'sql_compressed' doesn't have a default value |
    +---------+------+-----------------------------------------------------+
    2 rows in set (0.00 sec)

We have 2 warnings here; the first one is easily taken care for by providing a default value:

    ALTER TABLE `tmp.sql_log`
    CHANGE `id_ex` `id_ex` bigint(20) unsigned NOT NULL DEFAULT '0' AFTER `tmstmp`
    PARTITION BY HASH(DAY(tmstmp) % 7) PARTITIONS 7;

The second one is equally easily avoided by providing an empty value:

    M:7727678 [tmp]>insert into tmp.sql_log (tmstmp, sql_compressed) values ('2018-02-26 12:59:00', '');
    Query OK, 1 row affected (0.00 sec)

Let's look at the result at operation system level:

    / #  ls -la /var/lib/mysql/tmp/sql*.MYD
    -rw-rw----    1 mysql    mysql        56460 Feb 28 13:31 /var/lib/mysql/tmp/tmp.sql_log#P#p0.MYD
    -rw-rw----    1 mysql    mysql            0 Feb 28 13:31 /var/lib/mysql/tmp/tmp.sql_log#P#p1.MYD
    -rw-rw----    1 mysql    mysql            0 Feb 28 13:31 /var/lib/mysql/tmp/tmp.sql_log#P#p2.MYD
    -rw-rw----    1 mysql    mysql            0 Feb 28 13:31 /var/lib/mysql/tmp/tmp.sql_log#P#p3.MYD
    -rw-rw----    1 mysql    mysql            0 Feb 28 13:31 /var/lib/mysql/tmp/tmp.sql_log#P#p4.MYD
    -rw-rw----    1 mysql    mysql           20 Feb 28 13:31 /var/lib/mysql/tmp/tmp.sql_log#P#p5.MYD
    -rw-rw----    1 mysql    mysql           20 Feb 28 13:31 /var/lib/mysql/tmp/tmp.sql_log#P#p6.MYD

Beautiful. That's what I wanted to see.

Again, a stored procedure or a cron job could take care of table maintenance. 

    p_no=$(($(($(date "+%d") + 1)) % 7)) && docker exec m1 mysql -e "ALTER TABLE tmp.sql_log TRUNCATE PARTITION p$p_no"

Let me explain. The second part is self-explanatory:

    docker exec m1 mysql -e "ALTER TABLE tmp.sql_log TRUNCATE PARTITION p$p_no"

The first part defines the variable `p_no`. 

p_no=$(($(($(date "+%d") + 1)) % 7))

`$(date "+%d")` gives the date number. We add 1 by the expression `$(($(date "+%d") + 1))` and then we calculate the modulus by the expression `$(($(($(date "+%d") + 1)) % 7))`. That's fine.

Digression: Partitioning by day of week <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

But now it is obvious that the whole construction is screwed up. We work by the `day of month` number and have to take the modulus, so we will not cycle evenly to all of the partitions. We should not take the `day of month number` but the `day of week` number:

    M:7727678 [tmp]>>SELECT @dom:=DAY(NOW()),  @dom % 7;
    +------------------+----------+
    | @dom:=DAY(NOW()) | @dom % 7 |
    +------------------+----------+
    |               28 |        0 |
    +------------------+----------+
    1 row in set (0.00 sec)
    
    M:7727678 [tmp]>SELECT DAYOFWEEK(NOW());
    +------------------+
    | DAYOFWEEK(NOW()) |
    +------------------+
    |                4 |
    +------------------+
    1 row in set (0.00 sec)

So we have to reorganize our partitions. But before doing that, let's have a look at what we think the files should look like. To this end, I have manipulated the file-time of our data files:

    $ sudo touch -d 201802271211 /d/data/master/tmp/sql_log#P#p6.MYD
    $ sudo touch -d 201802261211 /d/data/master/tmp/sql_log#P#p5.MYD
    $ sudo touch -d 201802251211 /d/data/master/tmp/sql_log#P#p4.MYD
    $ sudo touch -d 201802241211 /d/data/master/tmp/sql_log#P#p3.MYD
    $ sudo touch -d 201802231211 /d/data/master/tmp/sql_log#P#p2.MYD
    $ sudo touch -d 201802221211 /d/data/master/tmp/sql_log#P#p1.MYD
    
    
    $ ls -latr  /d/data/master/tmp/sql_log#*.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 22 12:11 /d/data/master/tmp/sql_log#P#p1.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 23 12:11 /d/data/master/tmp/sql_log#P#p2.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 24 12:11 /d/data/master/tmp/sql_log#P#p3.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 25 12:11 /d/data/master/tmp/sql_log#P#p4.MYD
    -rw-rw----    1 dockrema dockrema        20 Feb 26 12:11 /d/data/master/tmp/sql_log#P#p5.MYD
    -rw-rw----    1 dockrema dockrema        20 Feb 27 12:11 /d/data/master/tmp/sql_log#P#p6.MYD
    -rw-rw----    1 dockrema dockrema    112920 Feb 28 14:58 /d/data/master/tmp/sql_log#P#p0.MYD

From here we see easily which partition is the oldest and should be truncated. 

You may wonder about the group and the owner of these files. In order to have permanent data, we map a directory structure from the host to the container `/d/data/master:/var/lib/mysql` (see volume definition <a href="#volumes">above</a>), so we look at the same data from 2 different positions. 

We manipulate these files from inside the docker container via group and owner `mysql`. If you look at these files from inside the container, you will see `mysql mysql`. 

    / # ls -latr  /var/lib/mysql/tmp/sql_log#*.MYD
    -rw-rw----    1 mysql mysql         0 Feb 22 12:11 /d/data/master/tmp/sql_log#P#p1.MYD
    -rw-rw----    1 mysql mysql         0 Feb 23 12:11 /d/data/master/tmp/sql_log#P#p2.MYD
    -rw-rw----    1 mysql mysql         0 Feb 24 12:11 /d/data/master/tmp/sql_log#P#p3.MYD
    -rw-rw----    1 mysql mysql         0 Feb 25 12:11 /d/data/master/tmp/sql_log#P#p4.MYD
    -rw-rw----    1 mysql mysql        20 Feb 26 12:11 /d/data/master/tmp/sql_log#P#p5.MYD
    -rw-rw----    1 mysql mysql        20 Feb 27 12:11 /d/data/master/tmp/sql_log#P#p6.MYD
    -rw-rw----    1 mysql mysql    112920 Feb 28 14:58 /d/data/master/tmp/sql_log#P#p0.MYD

But here I am looking from the host, and docker somehow introduces a group and a user `dockrema` to cope with this inside-outside view. That's all I know. Most probably there will be some more to explain and understand, but that's enough for me.

We need a similar function given by the shell.

    $  date +%w
    3

That's correct. Let's reorganize our partition now. Unfortunately I was misled by the naming `REORGANIZE PARTITION`. You cannot reorganize the whole scheme with that statement, but only one single partition. So the way to choose is

    M:7727678 [tmp]>ALTER TABLE `sql_log` REMOVE PARTITIONING;
    Query OK, 310 rows affected (0.03 sec)
    Records: 310  Duplicates: 0  Warnings: 0

    M:7727678 [tmp]>ALTER TABLE `sql_log` PARTITION BY HASH (DAYOFWEEK(tmstmp) % 7) PARTITIONS 7;
    Query OK, 310 rows affected (0.02 sec)
    Records: 310  Duplicates: 0  Warnings: 0

You see, in the meantime I issued other data changing operations so we now have 310 records. Let's inspect the file data. 

    $ ls -latr  /d/data/master/tmp/sql_log#*.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:08 /d/data/master/tmp/sql_log#P#p6.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:08 /d/data/master/tmp/sql_log#P#p5.MYD
    -rw-rw----    1 dockrema dockrema    112920 Feb 28 16:08 /d/data/master/tmp/sql_log#P#p4.MYD
    -rw-rw----    1 dockrema dockrema        20 Feb 28 16:08 /d/data/master/tmp/sql_log#P#p3.MYD
    -rw-rw----    1 dockrema dockrema        20 Feb 28 16:08 /d/data/master/tmp/sql_log#P#p2.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:08 /d/data/master/tmp/sql_log#P#p1.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:08 /d/data/master/tmp/sql_log#P#p0.MYD

Great surprise here. Why are all our records of the day in partition `p4`? shouldn't they be in `p3`? 

Well, the database starts with 1 for Sunday -- at least the shell and the database agree on the day to start with. Partitioning also starts with 0, so let's change our partitioning scheme again (remove the old one first):

    M:7727678 [tmp]>ALTER TABLE `sql_log` PARTITION BY HASH ((DAYOFWEEK(tmstmp) % 7) -1) PARTITIONS 7;
    Query OK, 310 rows affected (0.03 sec)
    Records: 310  Duplicates: 0  Warnings: 0

Now we should see what we want:

    $ ls -latr  /d/data/master/tmp/sql_log#*.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:14 /d/data/master/tmp/sql_log#P#p6.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:14 /d/data/master/tmp/sql_log#P#p5.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:14 /d/data/master/tmp/sql_log#P#p4.MYD
    -rw-rw----    1 dockrema dockrema    112920 Feb 28 16:14 /d/data/master/tmp/sql_log#P#p3.MYD
    -rw-rw----    1 dockrema dockrema        20 Feb 28 16:14 /d/data/master/tmp/sql_log#P#p2.MYD
    -rw-rw----    1 dockrema dockrema        20 Feb 28 16:14 /d/data/master/tmp/sql_log#P#p1.MYD
    -rw-rw----    1 dockrema dockrema         0 Feb 28 16:14 /d/data/master/tmp/sql_log#P#p0.MYD

The shell script for regularly truncating the oldest partition now reads

    p_no=$(($(($(date "+%w") + 1)) % 7)) && docker exec m1 mysql -e "ALTER TABLE tmp.sql_log TRUNCATE PARTITION p$p_no"
    
or rather   

    p_no=$(($(($(date "+%w"))) % 7)) && docker exec m1 mysql -e "ALTER TABLE tmp.sql_log TRUNCATE PARTITION p$p_no"

when you run this script at midnight. The first action will be to clear the partition which will be filled for the day with new data.

Digression: Inspecting the SQL log <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

The function `_sql_log_record` responsible for logging data changing actions must filter several commands which, although not changing any data, have to be sent to the master but should not be logged nevertheless because they don't add anything to our understanding. These are `USE`, `SHOW`, `SET`.

The result is a very nice list of all the actions the program performs on the database. If you format your SQL statements by line break, you can read them better as seen above, where the comment appears on a new line instead of at the end of the whole query.

Why did I take the pain in the first place? Well, unfortunately things don't work as I thought. When I start with a clean system and I launch this relatively simple action, my system is out of sync immediately.

The replication monitor (to which I added `Seconds_Behind_Master`) tells me everything is okay. 

    /path_to_your_script/mysql_repl_monitor.sh 240 ---------------------------------------------------- 2018-02-28_22:49:00
    175 =====> s1 OK 2018-02-28_22:49:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000001 Read_Master_Log_Pos 437588
    175 =====> s2 OK 2018-02-28_22:49:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000001 Read_Master_Log_Pos 437588

But the synchronizing script tells me otherwise:

    docker@boot2docker:/path_to_your_script$ ./mysql_rsync_yaws.sh
        =========================================== 2018-02-28_22:50:23
    41: " -------- FLUSH TABLES LOCK TABLES WRITE" DATE
    48: " -------- rsync datm dat1"
    sending incremental file list
    cmp_ex_tn#P#p6.MYD
    cmp_ex_tn#P#p6.MYI
    cmp_temp.MYD
    cmp_temp.MYI
    ex.MYD
    ex.MYI
    tn_de#P#p6.MYD
    tn_de#P#p6.MYI
    
    sent 192,358,460 bytes  received 172 bytes  29,593,635.69 bytes/sec
    total size is 4,469,961,199  speedup is 23.24
        =========================================== 2018-02-28_22:50:29
    sending incremental file list
    cmp_ex_tn#P#p6.MYD
    cmp_ex_tn#P#p6.MYI
    cmp_temp.MYD
    cmp_temp.MYI
    ex.MYD
    ex.MYI
    tn_de#P#p6.MYD
    tn_de#P#p6.MYI
    
    sent 192,358,460 bytes  received 172 bytes  54,959,609.14 bytes/sec
    total size is 4,469,961,199  speedup is 23.24
        =========================================== 2018-02-28_22:50:32
    59: " -------- SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1"
    66: " UNLOCK TABLES FLUSH TABLES"
        =========================================== 2018-02-28_22:50:35
    75: " -------- done" DATE
    ---------------------------------------------- time taken 12 seconds

Here you see that the ID plays a significant role. All partitioned tables were only touched in the partition belonging to ID 6. What happens here?

In order to find out I used the SQL logging table, because the binlog information didn't help me much. I must believe that the statements recorded in the logging table are written to the binlog and then read by the slaves, copied to their relay log and processed from there. The SQL logging table doesn't reveal anything unusual. Again: What happens here?

What does it mean when rsync thinks a file is different? In my understanding there should be some byte difference in both files. And if so, shouldn't this difference be reflected in the data?

    root@boot2docker:~# ls -la /d/data/master/dj5/cmp_ex_tn#P#p6.MYD
    -rw-rw----    1 dockrema dockrema    142740 Feb 28 23:10 /d/data/master/dj5/cmp_ex_tn#P#p6.MYD
    root@boot2docker:~# ls -la /d/data/slave1/dj5/cmp_ex_tn#P#p6.MYD
    -rw-rw----    1 dockrema dockrema    142740 Feb 28 21:19 /d/data/slave1/dj5/cmp_ex_tn#P#p6.MYD

Now this is revealing, isn't it? Both files have different file time data. How come?

The file on the master is written to, so it reflects the new file date. It is written to because the original record has been deleted and a new one has been inserted.

Exactly this mechanism should have happened on the slave as well. Therefore the data file of the slave should have been updated accordingly.

Let's see what we have got:

    M:7727678 [dj5]>desc cmp_ex_tn;
    +-------------+-----------------------+------+-----+-------------------+-----------------------------+
    | Field       | Type                  | Null | Key | Default           | Extra                       |
    +-------------+-----------------------+------+-----+-------------------+-----------------------------+
    | id_ex       | bigint(20) unsigned   | NO   | PRI | NULL              |                             |
    | lg          | varchar(2)            | NO   | PRI |                   |                             |
    | str         | mediumblob            | NO   |     | NULL              |                             |
    | tn_cnt      | mediumint(8) unsigned | NO   |     | 0                 |                             |
    | ca_tmstmp   | timestamp             | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
    +-------------+-----------------------+------+-----+-------------------+-----------------------------+
    5 rows in set (0.00 sec)

We have a timestamp here. That's another habit of mine. Most every table has an auto increment column and a timestamp. We can use that here.

    M:7727678 [dj5]>SELECT id_ex, ca_tmstmp FROM cmp_ex_tn WHERE id_ex = 6;
    +-------+---------------------+
    | id_ex | ca_tmstmp           |
    +-------+---------------------+
    |     6 | 2018-03-01 00:10:56 |
    +-------+---------------------+
    1 row in set (0.00 sec)

Now let us start the mysql client for the first slave:

     docker exec -it s1 mysql dj5

and issue the same command here:

    S:8728244 [dj5]>SELECT id_ex, ca_tmstmp FROM cmp_ex_tn WHERE id_ex = 6;
    +-------+---------------------+
    | id_ex | ca_tmstmp           |
    +-------+---------------------+
    |     6 | 2018-03-01 00:10:56 |
    +-------+---------------------+
    1 row in set (0.00 sec)

Same procedure for the second slave:

    S:8715945 [dj5]>SELECT id_ex, ca_tmstmp FROM cmp_ex_tn WHERE id_ex = 6;
    +-------+---------------------+
    | id_ex | ca_tmstmp           |
    +-------+---------------------+
    |     6 | 2018-03-01 00:10:56 |
    +-------+---------------------+
    1 row in set (0.00 sec)

or, more compact:


    $ docker exec m1 mysql -e "SELECT 'm1', id_ex, ca_tmstmp FROM dj5.cmp_ex_tn WHERE id_ex = 6" && \
      docker exec s1 mysql -e "SELECT 's1', id_ex, ca_tmstmp FROM dj5.cmp_ex_tn WHERE id_ex = 6" && \
      docker exec s2 mysql -e "SELECT 's2', id_ex, ca_tmstmp FROM dj5.cmp_ex_tn WHERE id_ex = 6"
    m1      id_ex   ca_tmstmp
    m1      6       2018-03-01 00:10:56
    s1      id_ex   ca_tmstmp
    s1      6       2018-03-01 00:10:56
    s2      id_ex   ca_tmstmp
    s2      6       2018-03-01 00:10:56

Well, this is perfect, isn't it? Why does rsync think otherwise? 

Here my memory fails me. Somewhere in one of my scripts I used the term `diff`. I remember I investigated the question of the best methods to find if 2 files are identical a couple of days ago. And I remember that this question had no easy answer, and I chose `diff` as being the most reliable.

Now I need something like

    grep -rn '/mnt/sda1/xq/dj/application/' -e '' 

for a totally different directory: `path_to_your_script`. Numerous times I have overwritten this template by typing, but now I will define a new hotkey or rather shortcut.

    ::ggrp::grep -rn '/path_to_your_script/' -e ''

I don't find anything. Now I remember. `diff` was one of the candidates which were not good. It was something with `md5`. 

Digression: Comparing files <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

And here I have it: `md5sum`, and it is contained in a script named `mysql_cmp.sh`.  I have written the script a couple of days ago and I can't remember. I guess this is the result of having done something with utmost satisfaction, when as a result the mind puts things at rest.

This script compares all the files and takes action if the `md5sum` of files -- which should be equal -- is different. 

Looking at the script, the first thing I see is that it is triggered by a file. This file should have been written to by another script which adds the name of the slave having problems. This file is read and reset, the list of lines is made unique, so the script is fed with the names of the slaves to take action on.

Obviously, the supervising script writing the trigger file will be called much more often than this script. Or the other way around: I don't take action the moment I notice a problem, but only in much bigger intervals because the repair action may take much longer. Otherwise I might call the repair script again and again while it is still running with unknown consequences. 

If the trigger file is empty, no action is taken at all. So this script only knows about slaves, not about files which are different. This is what the script tries to find out.

To this end, the script takes all data files one by one. For each file, the master is called to flush that table and lock it for write. Then the same is done with each slave, so that we should have comparable conditions. And then the data files of the master and the slave in question are compared.

Then the lock on the slave is relieved, the next slave is done the same way, and when all slaves are done the master releases the lock on this table to repeat the whole process for the next table.

This way, write table locks are minimized in contrast to the rsync method, where all tables are locked for writing on the master as long as the whole process takes.

The function doing the actual compare job is defined like this:

    # we check master file
    chk_master=$(sudo md5sum $datm/dj5/$2.MYD | awk -F" " '{print $1}')
    # we check slave file
    chk_slave=$(sudo md5sum /d/data/slave$no/dj5/$2.MYD | awk -F" " '{print $1}')

In case these 2 expressions are different, the master file is copied to the slave file.

What about using this mechanism to shed light on this enigmatic situation?

    $ sudo md5sum /d/data/master/dj5/cmp_ex_tn#P#p6.MYD | awk -F" " '{print $1}'
    1e17b1d93fb117bc8f0408259e4433e3
    $ sudo md5sum /d/data/slave1/dj5/cmp_ex_tn#P#p6.MYD | awk -F" " '{print $1}'
    896cffca7c7252ef1b043ecfa1137a5e

Well, it looks like these files are indeed different. 

So the conclusion here should be to start with a clean setup which can be achieved either way by `mysql_rsync_lock.sh` or `mysql_cmp.sh`. The slaves take care of errors. We rely on and trust the slave status information. 

Monitor this action with `mysql_repl_monitor.sh` and write a trigger file in case of an error. Then let cron regularly check if there are triggers, and if so, take action.

As I already have a script which is run every 30 minutes anyway, I integrated the call here:

    # will check slaves for integrity and copy -- triggered by /tmp/repl_cmp.trigger set by /path_to_your_script/mysql_repl_monitor.sh
    /path_to_your_script/mysql_cmp.sh            

I think there is room for one improvement here. In case the next error occurs, I will have a closer look to the error message. If the error message tells me which table has problems, I could immediately take action on this table without checking all the others. To this end I have to see the error message because I have to extract the table name from that string.

Or ask Google: [MariaDB Error Codes](https://mariadb.com/kb/en/library/mariadb-error-codes/). There are seem to be 2 types of error messages which may apply here:

    1004	HY000	ER_CANT_CREATE_FILE	Can't create file '%s' (errno: %d)
    1005	HY000	ER_CANT_CREATE_TABLE	Can't create table '%s' (errno: %d)

It shouldn't be too hard to get the table name from this information. 

Digression: Why are data files different <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Still, I feel uneasy about the situation. In my understanding those files should be identical. Of course, depending on the structure of the disc, this data could be spread out over totally different sectors and whatnot, but byte-wise they should be equal.

Are they equal database-wise? That question should be easy to answer by mysqldump.

    $ docker exec m1 /bin/ash -c 'mysqldump --opt --compact dj5 cmp_ex_tn > /tmp/cmp_ex_tn_m1.sql'
    $ docker exec s1 /bin/ash -c 'mysqldump --opt --compact dj5 cmp_ex_tn > /tmp/cmp_ex_tn_s1.sql'
    $ docker exec s2 /bin/ash -c 'mysqldump --opt --compact dj5 cmp_ex_tn > /tmp/cmp_ex_tn_s2.sql'

At first glance, it looks good.

    $ ls -latr /tmp/cmp_ex*
    -rw-r--r--    1 root     root      15959336 Mar  1 10:29 cmp_ex_tn_m1.sql
    -rw-r--r--    1 root     root      15959336 Mar  1 10:30 cmp_ex_tn_s1.sql
    -rw-r--r--    1 root     root      15959336 Mar  1 10:30 cmp_ex_tn_s2.sql

But even thorough inspection reveals that they are indeed identical.

    $ md5sum /tmp/cmp_ex_tn_m1.sql | awk -F" " '{print $1}'
    42d3702f3e0e7f2f6956d9a722f1eb88
    $ md5sum /tmp/cmp_ex_tn_s1.sql | awk -F" " '{print $1}'
    42d3702f3e0e7f2f6956d9a722f1eb88
    $ md5sum /tmp/cmp_ex_tn_s2.sql | awk -F" " '{print $1}'
    42d3702f3e0e7f2f6956d9a722f1eb88

Any explanation why the data files are different? No idea.

Accidentally I had a look at the monitoring file and noticed that first slave s1 and then the slave s2 had a lag, which was taken care of automatically, as it seems: 

    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:23:00
    175 =====> s1 OK 2018-03-12_12:23:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24624489
    175 =====> s2 OK 2018-03-12_12:23:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24624489
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:24:00
    175 =====> s1 OK 2018-03-12_12:24:00 Seconds_Behind_Master 31 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24658308
    175 =====> s2 OK 2018-03-12_12:24:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24658308
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:25:00
    175 =====> s1 OK 2018-03-12_12:25:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24694252
    175 =====> s2 OK 2018-03-12_12:25:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24694252
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:26:00
    175 =====> s1 OK 2018-03-12_12:26:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24700335
    175 =====> s2 OK 2018-03-12_12:26:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24700335
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:27:00
    175 =====> s1 OK 2018-03-12_12:27:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24706366
    175 =====> s2 OK 2018-03-12_12:27:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24706366
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:28:00
    175 =====> s1 OK 2018-03-12_12:28:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24760084
    175 =====> s2 OK 2018-03-12_12:28:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24760084
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:29:00
    175 =====> s1 OK 2018-03-12_12:29:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24760084
    175 =====> s2 OK 2018-03-12_12:29:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24760084
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:30:00
    175 =====> s1 OK 2018-03-12_12:30:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24788628
    175 =====> s2 OK 2018-03-12_12:30:00 Seconds_Behind_Master 2 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 24788628
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:31:00
    175 =====> s1 OK 2018-03-12_12:31:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 25886547
    175 =====> s2 OK 2018-03-12_12:31:00 Seconds_Behind_Master 62 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 25886547
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:32:00
    175 =====> s1 OK 2018-03-12_12:32:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 25901745
    175 =====> s2 OK 2018-03-12_12:32:00 Seconds_Behind_Master 122 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 25901745
    
    /c/bak/mysql_repl_monitor.sh 240 ------------------------------------------------------------------- 2018-03-12_12:33:00
    175 =====> s1 OK 2018-03-12_12:33:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 25939553
    175 =====> s2 OK 2018-03-12_12:33:00 Seconds_Behind_Master 0 Master_Log_File mysql-bin.000004 Read_Master_Log_Pos 25939553

Looks like I have no chance to find out what happened here.

Digression: Automatic git checkout <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Looking at my crontab file, I notice a nice service running every minute which not only saves my work but eases my life as well. I let this script checkout all my work every minute. 

    # will do automatic git entries
    * * * * * /path_to_your_script/git_auto.sh

This script has another interesting feature. In regular intervals I run a PHP script which produces `CREATE TABLE` statements for all tables in the database. This file is called `_show_create_table.sql` and is put under revision control. 

Table definitions do change from time to time, and it is error prone to rely on manually taking notes.

    #FILE=git_auto.sh
    #!/bin/sh
    
    TIMEDIFF=7200
    #TIMEDIFF=0
    
    DATE=$(date -u --date="@$(($(date -u +%s) + $TIMEDIFF))" "+%Y-%m-%d %H:%M:%S")
    
    echo --- $DATE
    
    cp /tmp/_show_create_table.sql /c/xq/dj/
    
    cd /c/xq/dj
    /usr/local/bin/git commit -am "$DATE"

The last line shows the recipe. You change to whatever directory you have under revision control, and then simply call that statement. 

In case I have to switch back and look for a clean situation, with the shortcut `ggl` the git log is called, made pretty:

    $ git log --oneline | tr '\\n' | more
    775fb82 2018-02-28 23:20:00
    eb7b36a 2018-02-28 23:01:00
    26618e6 2018-02-28 13:10:00
    3b684a7 2018-02-28 13:09:00
    ed87027 2018-02-28 13:08:00
    6eed4ae 2018-02-28 13:01:01
    2443766 2018-02-27 18:04:00
    61135a3 2018-02-27 18:02:00
    f38f4ca 2018-02-27 16:44:00
    8de7f38 2018-02-27 16:43:00
    1bad3ed 2018-02-27 16:42:00
    0856873 2018-02-27 16:41:00
    f127310 2018-02-27 16:40:00
    a1a6ada 2018-02-27 16:39:00
    090f11f 2018-02-27 16:38:00
    5cc2ae1 2018-02-27 14:55:00
    5a04eb0 2018-02-27 14:54:00
    4907dae 2018-02-27 11:44:00
    cffb662 2018-02-27 11:43:00
    4b65372 2018-02-27 11:42:00
    9fb094e 2018-02-26 22:58:00
    53b5467 2018-02-26 22:56:00
    4214095 2018-02-26 22:50:00
    3106130 2018-02-26 19:52:00
    b60a14d 2018-02-26 19:51:00
    01e9672 2018-02-26 18:51:00
    47859fa 2018-02-26 18:46:01
    f792085 2018-02-26 18:45:00
    8a5cabf 2018-02-26 18:44:00
    5c57c31 2018-02-26 18:37:00
    49d6bba 2018-02-26 18:33:00
    82ba3fa 2018-02-26 18:32:00
    74c2eb6 2018-02-26 18:31:00
    a6f84fc 2018-02-26 18:30:00
    4a75434 2018-02-26 18:29:00
    c49a3fe 2018-02-26 18:28:00
    262270c 2018-02-26 18:23:00
    1d9fef5 2018-02-26 18:22:00
    dbff348 2018-02-26 18:18:00
    2313708 2018-02-26 18:12:00
    62edf55 2018-02-26 18:11:00
    c57a861 2018-02-26 18:10:00
    7ef61a6 2018-02-26 18:08:00
    4e455b6 2018-02-26 18:07:00
    05f63da 2018-02-26 18:06:00
    6d5d1f7 Merge branch 'master' into temp4
    d99f57e 2018-02-26 17:58:00
    3aac429 2018-02-26 17:37:00
    46680a3 2018-02-26 17:36:00
    d3e7c20 2018-02-26 17:35:00
    24bad4f 2018-02-26 17:26:00
    b74e7cb 2018-02-26 17:23:00
    d41b8ac 2018-02-26 17:19:00
    8fcc547 2018-02-26 17:18:00
    --More--

Interesting enough, you can see when I was asleep or at least didn't change any of the files under control for some length of time. 

At the end you see the message from pushing the whole stuff to origin or rather master back to the working branch. Maybe this is the answer I was looking for: by merging back I get an automatic label.

In case I have screwed things up somewhere along the lines without noticing and have to find the last clean checkout, I simply check out different revisions by jumping in this list and picking the one in the middle until I have what I want. This procedure is very fast and guaranteed to succeed.

The only prerequisite is that this command runs continuously. It happened that it didn't due to some git error:

    docker@boot2docker:/c/xq/dj/$ git checkout temp7
    fatal: Unable to create '/c/xq/dj//.git/index.lock': File exists.
    
    Another git process seems to be running in this repository, e.g.
    an editor opened by 'git commit'. Please make sure all processes
    are terminated then try again. If it still fails, a git process
    may have crashed in this repository earlier:
    remove the file manually to continue.

So how to prevent this? Create another supervising script? I don't know. It happened to me this morning and I thought I would have to get back to work done hours ago which unfortunately hasn't been checked out due to this error. Luckily, this turned out to be false, so I could just carry on by deleting this file `.git/index.lock`. But otherwise this situation wouldn't have been so nice with 12 files modified. It's no fun to re-create things you have done before.

Of course, this simple and primitive automatic procedure is possible only because I work alone. I have no experience with teamwork since the 90s, when revision control didn't exist yet. This condition may change, though, and if so I hope to profit from the experience of my collaborators.

Digression: Automatic boot2docker setup <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

One more thing that I had been struggling with very long until I found a good solution: If you work with boot2docker, at reboot you will lose all data which is not saved at some safe place. In particular, crontab data and the home directory `/home/docker` is lost. Of course, data in the `tmp` directory is lost as well, but that's to be expected and rather nice.

There is one place where you can manipulate the startup behavior:

    /var/lib/boot2docker/profile

It is inconvenient to manipulate this file directly (remember this path and be root), so I only use it to call another script from my `path_to_your_script` directory called `up.sh` where I put all my instructions to instead.

    #FILE=/var/lib/boot2docker/profile
    #!/bin/sh
    
    if [ ! -d "/c" ]; then
        mkdir /c
        mount /dev/sda1 /c
        /bin/sh /path_to_your_script/up.sh
    else
        /bin/sh /path_to_your_script/docker_stop.sh
    fi

This script is called at shutdown also and the `else` part will make sure that the whole setup is shut down cleanly. In particular the database engine may not like to be suddenly killed. `docker_stop.sh` calls the proper docker shutdown procedure. This way you're safe.

In starting, the script `up.sh` will call the script `cron_build.sh` which populates the crontab file based on the template `cron.sh`. 

    #!/bin/ash
    FILE=/path_to_your_script/cron_build.sh
    # we call this at startup via /mnt/sda1/var/lib/boot2docker/profile & /path_to_your_script/up.sh
    
    while read LINE
    do
        (crontab -l 2>/dev/null; echo "$LINE")| crontab -
    done < /path_to_your_script/cron.sh                                          

So whenever I change something in my crontab, in order to make it permanent, I have to change the crontab template `cron.sh` accordingly. But that's it. With this setup, it is absolutely safe to say `sudo reboot` to my docker machine, provided the USB stick is in place. After boot up, it is safe to remove the USB stick in order to not accidentally break it if it sits in a front port. 

End of digression.

Why roll your own, revisited <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Some more considerations might be helpful. Let's again learn by example.
 
For the problem of slaves running out of sync, I already mentioned the ready-to-use solutions by "Experts in Database Performance Management" Percona ([pt-table-checksum](https://www.percona.com/doc/percona-toolkit/LATEST/pt-table-checksum.html), [pt-table-sync](https://www.percona.com/doc/percona-toolkit/LATEST/pt-table-sync.html)), but those make heavy use of Perl which may not be at your disposal and you may not be willing to install a big software packet just to connect to your database engine to begin with. So this is of no use for you. 

But there is another reason why to step back here. I concede that, at first glance, it's compelling to be lazy and use other people's programs, but you have to understand them, too, if you want to make use of them, and if these tools don't work out as expected, you will have to analyze their code anyway (if you can) and try to make sense of it and find that piece of code that doesn't do as it should (I am no expert in Perl and would not like to invest time and energy to become good enough to debug a program by Percona). 

A second example: The shell script of Giuseppe Maxia mentioned above didn't record the complete error message, but only the first word, hence his e-mail doesn't have any information about the nature of the error, as the first word in the error message is "Error", which doesn't help at all. 

Too bad and easily fixed by adding a second function especially for use in the e-mail or the log file extracting not only a single value from the response of the server but the complete sentence to be used for those error messages. 

But he also uses arrays, which are handy in bash, albeit not implemented in ash. So for this reason alone his code is not usable out-of-the-box and has to be rewritten for platforms not having bash.

Nevertheless, it is great to build upon brilliant code of masters like Giuseppe Maxia, learn from them and grow along the way. Which is exactly the mission of Stack Overflow, if I understand correctly. 

Thanks a lot to everybody involved in this great and wonderful worldwide endeavor. Who would have been able to dream of this phenomenon, say, 30 years ago?

Summarizing, if you roll your own solution, 

 - you know what you're doing,  

 - your solution reflects exactly the nature of your special setup,  

 - and you're in complete control.

Again you will be glad to have a handy debugging tool like `echo_line_no` with complex tasks like database replication repair on platforms without LINENO. You will need it even more so as your solution cannot claim to be tested by a plethora of experts with all kinds of field experience.

Have fun <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Finally, if you want to repeat the above given test `test_echo_line_no.sh`, don't forget to replace `path_to_your_script` with your own path (or create a symbolic link, whichever is easier for you). Take this sample as a starting point for your own creativity. Maybe there are other exciting things you can do with this approach I couldn't come up with (yet). 

You see, I haven't tested this approach under heavy conditions because unfortunately I already had developed these database repair scripts mentioned above without making use of `echo_line_no` -- in fact the deficiencies in debugging these programs finally made me look for a solution. I reckoned with lots of people having the same problem and some who not only know what to do, but published solutions on StackOverflow, for example.

I finally found this page via Google "shell script display line numbers -diff -tail"; the term "shell script display line numbers -bash -diff -tail" which I used before obviously missed this entry. 

The contributions to this page so far are nearly 5 years old now. They have shown me that there is no solution for my problem except I create one myself, which I did.

Digression: Proof of concept <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

I reorganized one of my old scripts to see if everything works as expected. The purpose is to start complex operations not from the browser but via shell allowing to let several of those operations run concurrently. The same process can be started from the browser as well, but then the first process is visible from the browser, the others are also run in a shell.

Now with Docker this is a bit complicated as the browser process is handled by a container, and this container cannot do what the process needs. So I write a trigger file with the appropriate shell instruction in a directory accessible by the container and the shell (`tmp`), and let Cron check for the existence of this file and eventually start the processes recorded there. 

These are the 2 debugging instructions I inserted in my code:

    echo_line_no "==do== ID_EX :$ID_EX: FILE :$FILE: Date :$DATE:  
    >>>>>>> : ==do== CMD :curl -N -s \"$CMD\":" 

(a multiline comment) and 
    
    echo_line_no "== GOOD!!! ID_EX :$ID_EX: =================== used :$USED: secs " DATE      

The result is beautiful, much better than all of these echoes I used before.

    docker@boot2docker:/mnt/sda1/tmp$ nohup /path_to_your_script/tsm3.sh "6" </dev/null &>/dev/null &
        78       "==do== ID_EX :$ID_EX: FILE :$FILE: Date :$DATE:
        >>>>>>> : ==do== ID_EX :6: FILE :tsmst.sh: Date :2018-03-02_22:00:51:
        >>>>>>> : ==do== CMD :curl -N -s "localhost:8342/qh/6?srt=1&d=1&bak=1&lg=en":
        97           "== GOOD!!! ID_EX :$ID_EX: =================== used :$USED: secs " DATE
        >>>>>>> : == GOOD!!! ID_EX :6: =================== used :19: secs
        =========DATE======== :2018-03-02_22:01:10:
        
The output is even readable when those processes are intertwined:

    docker@boot2docker:/mnt/sda1/tmp$ nohup /path_to_your_script/tsm3.sh "6 359" </dev/null &>/dev/null &
        78       "==do== ID_EX :$ID_EX: FILE :$FILE: Date :$DATE:
        >>>>>>> : ==do== ID_EX :6: FILE :tsmst.sh: Date :2018-03-02_22:01:37:
        >>>>>>> : ==do== CMD :curl -N -s "localhost:8342/qh/6?srt=1&d=1&bak=1&lg=en":
    
        78       "==do== ID_EX :$ID_EX: FILE :$FILE: Date :$DATE:
        >>>>>>> : ==do== ID_EX :359: FILE :tsmst.sh: Date :2018-03-02_22:01:42:
        >>>>>>> : ==do== CMD :curl -N -s "localhost:8342/qh/359?srt=1&d=1&bak=1&lg=en":
        97           "== GOOD!!! ID_EX :$ID_EX: =================== used :$USED: secs " DATE
        >>>>>>> : == GOOD!!! ID_EX :6: =================== used :18: secs
        =========DATE======== :2018-03-02_22:01:55:
        97           "== GOOD!!! ID_EX :$ID_EX: =================== used :$USED: secs " DATE
        >>>>>>> : == GOOD!!! ID_EX :359: =================== used :29: secs
        =========DATE======== :2018-03-02_22:02:11:
    
You also see that it is important to know which script is doing what; the calling script `tsm3.sh` is different from the one shown in the output: `tsmst.sh`, given by the variable `FILE` defined by habit at the top of the script.

Later, I found that I wanted to get more information about the sequence of shell scripts, so I rewrote them accordingly. Now you can see exactly what calls what and what is happening when.

    tsm3.sh ------------------------------
        68       "FILE :$FILE: ID_EX :$ID_EX: LG :$LG: DEL :$DEL: CMD :$CMD:"
        >>>>>>> : FILE :/path_to_your_script/tsm3.sh: ID_EX :6: LG :en: DEL :1: CMD :/path_to_your_script/tsmst.sh 6 en 1:
    
    echo_line_no.sh ------------------------------
       114       "==do== CMD :curl -N -s \"$CMD\":"
        >>>>>>> : ==do== CMD :curl -N -s "localhost:8342/qh/6?srt=1&d=1&bak=1&lg=en":
       151           "== GOOD!!! ID_EX :$ID_EX: ========= LG :$LG: ========== used :$USED: secs " DATE
        >>>>>>> : == GOOD!!! ID_EX :6: ========= LG :en: ========== used :15: secs
        =========DATE======== :2018-03-13_12:55:03:

Digression: Record by database <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

That's alright, but not really good. We worked with 2 parameters here only, and both produced relatively short execution times. What about a long parameter list and really long execution times? We just wouldn't be able to interpret what's going on.

That's why we need a database solution here.

    M:7727678 [tmp]>SHOW CREATE TABLE tsmst\G
    *************************** 1. row ***************************
           Table: tsmst
    Create Table: CREATE TABLE `tsmst` (
      `id_ex` bigint(20) unsigned NOT NULL,
      `tmstmp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
      `comment` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
      KEY `id_ex` (`id_ex`)
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8
    1 row in set (0.00 sec)

In the script shell which manages a single parameter, first make sure we have no old entries and then insert the start data.

    docker exec m1 mysql -e "DELETE FROM tmp.tsmst WHERE id_ex = '$ID_EX'; 
        INSERT INTO tmp.tsmst VALUES ($ID_EX, NOW(), '$CMD')"

In case we have a problem, record this as well:    
    
    docker exec m1 mysql -e "INSERT INTO tmp.tsmst VALUES ($ID_EX, NOW(), '##### NO!!!!! ######### PROBLEM HERE')" 

Success is recorded likewise:

    docker exec m1 mysql -e "INSERT INTO tmp.tsmst VALUES ($ID_EX, NOW(), '== GOOD!!!======== used :$USED: secs ')" 

The database table shows similar data, but can be selected:

    M:7727678 [tmp]>SELECT * FROM tsmst;
    +-------+---------------------+-------------------------------------------------+
    | id_ex | tmstmp              | comment                                         |
    +-------+---------------------+-------------------------------------------------+
    |     6 | 2018-03-04 18:52:11 | localhost:8342/qh/6?srt=1&d=1&bak=1&lg=en       |
    |   359 | 2018-03-04 18:52:16 | localhost:8342/qh/359?srt=1&d=1&bak=1&lg=en     |
    |     6 | 2018-03-04 18:52:29 | == GOOD!!!======== used :18: secs               |
    |   359 | 2018-03-04 18:52:47 | == GOOD!!!======== used :31: secs               |
    +-------+---------------------+-------------------------------------------------+
    4 rows in set (0.00 sec)
    
    M:7727678 [tmp]>SELECT * FROM tsmst WHERE id_ex = '6';
    +-------+---------------------+-----------------------------------------------+
    | id_ex | tmstmp              | comment                                       |
    +-------+---------------------+-----------------------------------------------+
    |     6 | 2018-03-04 18:52:11 | localhost:8342/qh/6?srt=1&d=1&bak=1&lg=en     |
    |     6 | 2018-03-04 18:52:29 | == GOOD!!!======== used :18: secs             |
    +-------+---------------------+-----------------------------------------------+
    2 rows in set (0.00 sec)
    
    M:7727678 [tmp]>SELECT * FROM tsmst WHERE id_ex = '359';
    +-------+---------------------+-------------------------------------------------+
    | id_ex | tmstmp              | comment                                         |
    +-------+---------------------+-------------------------------------------------+
    |   359 | 2018-03-04 18:52:16 | localhost:8342/qh/359?srt=1&d=1&bak=1&lg=en     |
    |   359 | 2018-03-04 18:52:47 | == GOOD!!!======== used :31: secs               |
    +-------+---------------------+-------------------------------------------------+
    2 rows in set (0.00 sec)

The next sequence shows how the job is done sequentially until nothing is left,    
    
    M:7727678 [tmp]>SELECT id_ex, COUNT(comment) cnt FROM tsmst GROUP BY id_ex HAVING cnt =1;
    +-------+-----+
    | id_ex | cnt |
    +-------+-----+
    |     6 |   1 |
    |   359 |   1 |
    +-------+-----+
    2 rows in set (0.00 sec)
    
    M:7727678 [tmp]>SELECT id_ex, COUNT(comment) cnt FROM tsmst GROUP BY id_ex HAVING cnt =1;
    +-------+-----+
    | id_ex | cnt |
    +-------+-----+
    |   359 |   1 |
    +-------+-----+
    1 row in set (0.00 sec)
    
    M:7727678 [tmp]>SELECT id_ex, COUNT(comment) cnt FROM tsmst GROUP BY id_ex HAVING cnt =1;
    Empty set (0.00 sec)

Most important will be the selection for errors:

    M:7727678 [tmp]>SELECT * FROM tsmst WHERE comment LIKE '##### NO!!!!!';
    Empty set (0.00 sec)

That's much better than `grep`ing the log file.

Digression: More complexity by languages <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

We have seen more complexity and the samples suggest that there is even more to show. The next example shows our script working on different languages: in the case of ID `2181` the 5 languages `de` (German), `en` (English), `fr` (French), `nl` (Dutch), `zh` (Chinese). 

In order to get things right, I used this same table to record debug messages from my PHP program as well. This proved to be a very clever idea.

The mechanism starts with just one language and if there are other languages to process, trigger commands are written to a file (first 5 lines). 

Crontab runs a shell script every minute looking for the existence of this file, and if so, executes the commands in this file (next 4 lines) and moves that trigger file to a backup file for debug purposes.

If a language is processed (`tsmst.sh == GOOD!!!`), the data is transferred from the `tmp` database to the main database `dj5` (`done INSERT INTO`), the backup table in database `bak` is dropped (`DROP TABLE IF EXISTS`), at last the `tmp` result table is copied to the database `bak` in case the data should be of use for inspection (`INSERT INTO bak`). Of course, before doing that, the `bak` table has to be created, which is not recorded here (`CREATE TABLE $bak LIKE $tbl`). I didn't use that `bak` table for a long time now, so I guess I don't need it anymore and can safely drop this process.

If all languages are processed, another mechanism is invoked which will produce results files for each language (the last 5 lines). The whole protocol looks really nice and is extremely useful for debugging.

    M:7727678 [tmp]>SELECT tmstmp, comment FROM tsmst WHERE id_ex = '2181' ORDER BY 1, 2;
    +---------------------+---------------------------------------------------------------------------------------------------+
    | tmstmp              | comment                                                                                           |
    +---------------------+---------------------------------------------------------------------------------------------------+
    | 2018-03-09 02:12:14 | 24657 nohup /path_to_your_script/tsmst.sh 2181 de 0 2>&1 1>>/tmp/good.ts 0</dev/null 1>&/dev/null |
    | 2018-03-09 02:12:14 | 24662 nohup /path_to_your_script/tsmst.sh 2181 en 0 2>&1 1>>/tmp/good.ts 0</dev/null 1>&/dev/null |
    | 2018-03-09 02:12:14 | 24662 nohup /path_to_your_script/tsmst.sh 2181 fr 0 2>&1 1>>/tmp/good.ts 0</dev/null 1>&/dev/null |
    | 2018-03-09 02:12:14 | 24662 nohup /path_to_your_script/tsmst.sh 2181 nl 0 2>&1 1>>/tmp/good.ts 0</dev/null 1>&/dev/null |
    | 2018-03-09 02:12:14 | 24662 nohup /path_to_your_script/tsmst.sh 2181 zh 0 2>&1 1>>/tmp/good.ts 0</dev/null 1>&/dev/null |
    | 2018-03-09 02:13:01 | tsmst.sh INIT en DEL :0:                                                                          |
    | 2018-03-09 02:13:01 | tsmst.sh INIT fr DEL :0:                                                                          |
    | 2018-03-09 02:13:01 | tsmst.sh INIT zh DEL :0:                                                                          |
    | 2018-03-09 02:13:02 | tsmst.sh INIT nl DEL :0:                                                                          |
    | 2018-03-09 02:16:03 | tsmst.sh == GOOD!!!===== LG :fr: === used :182: secs                                              |
    | 2018-03-09 02:16:03 | 22380 while fr done, about to _transfer_tmp_to_dj5                                                |
    | 2018-03-09 02:16:03 | 24756 done INSERT INTO tn_fr                                                                      |
    | 2018-03-09 02:16:03 | 24798 DROP TABLE IF EXISTS bak.tn_2181_fr                                                         |
    | 2018-03-09 02:16:03 | 24808 INSERT INTO bak.tn_2181_fr SELECT * FROM tmp.tn_2181_fr                                     |
    | 2018-03-09 02:16:21 | tsmst.sh == GOOD!!!===== LG :en: === used :200: secs                                              |
    | 2018-03-09 02:16:21 | 22380 while en done, about to _transfer_tmp_to_dj5                                                |
    | 2018-03-09 02:16:21 | 24756 done INSERT INTO tn_en                                                                      |
    | 2018-03-09 02:16:21 | 24798 DROP TABLE IF EXISTS bak.tn_2181_en                                                         |
    | 2018-03-09 02:16:21 | 24808 INSERT INTO bak.tn_2181_en SELECT * FROM tmp.tn_2181_en                                     |
    | 2018-03-09 02:16:24 | tsmst.sh == GOOD!!!===== LG :zh: === used :203: secs                                              |
    | 2018-03-09 02:16:24 | 22380 while zh done, about to _transfer_tmp_to_dj5                                                |
    | 2018-03-09 02:16:24 | 24756 done INSERT INTO tn_zh                                                                      |
    | 2018-03-09 02:16:24 | 24798 DROP TABLE IF EXISTS bak.tn_2181_zh                                                         |
    | 2018-03-09 02:16:24 | 24808 INSERT INTO bak.tn_2181_zh SELECT * FROM tmp.tn_2181_zh                                     |
    | 2018-03-09 02:16:36 | tsmst.sh == GOOD!!!===== LG :nl: === used :215: secs                                              |
    | 2018-03-09 02:16:36 | 22380 while nl done, about to _transfer_tmp_to_dj5                                                |
    | 2018-03-09 02:16:36 | 24756 done INSERT INTO tn_nl                                                                      |
    | 2018-03-09 02:16:36 | 24798 DROP TABLE IF EXISTS bak.tn_2181_nl                                                         |
    | 2018-03-09 02:16:36 | 24808 INSERT INTO bak.tn_2181_nl SELECT * FROM tmp.tn_2181_nl                                     |
    | 2018-03-09 02:16:36 | 24832 de 2181 _build_tn                                                                           |
    | 2018-03-09 02:16:36 | 24832 en 2181 _build_tn                                                                           |
    | 2018-03-09 02:16:36 | 24832 fr 2181 _build_tn                                                                           |
    | 2018-03-09 02:16:36 | 24832 nl 2181 _build_tn                                                                           |
    | 2018-03-09 02:16:36 | 24832 zh 2181 _build_tn                                                                           |
    +---------------------+---------------------------------------------------------------------------------------------------+
    29 rows in set (0.00 sec)

You can see which line is written by the shell script tsmst.sh responsible for starting the language specific process, the rest stems from my PHP script preceded by the line number for easy identification. 

It's funny -- I'm programming for so long now and never developed a viable idea of how to track what a program is really doing. CodeIgniter has a `benchmark class` which I misused for this purpose at times, but that was unrewarding mostly. This idea, however, seems to be really helpful.

Digression: Dirty debugging techniques <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Of course, there are many debugging techniques. When I started with PHP, there was no `try except` mechanism, so I had to develop my own, basically with some kind of echo construction -- of course, what else? That's fine, I can show any kind of variable and any number of them plus any kind of arrays and objects. 

And if there should be some new desire, I'd just enhance my function. Actually, it is a small set of functions. In CodeIgniter, they reside in a helper file.

When exceptions were introduced in PHP, I wasn't convinced that this concept would give me any advantage, and since then I have seen many examples, but never made it a habit -- in fact I don't use them at all. They are fancy and they are cool, but I need ad hoc debugging techniques, and this is overkill. I didn't even use them back in times when I was programming in VC++ and Delphi, although they were highly recommended to me by a very gifted colleague.

But that's another story. PHP is an interpreted language, debugging is easy. With Delphi and VC++, being languages to be compiled first before you can see if there is an error, debugging was extremely time-consuming and boring just the same.

The same holds true with testing mechanisms. There are test units everywhere, but I don't use them. Either code is okay and it works under all circumstances, or it doesn't and I have to find out the conditions. In all these years, cases where some error would return were extremely seldom. And in these one or two cases the error conditions were so complex, it wouldn't have paid out to define a test case in the first place.

Things may be different when you publish open source code to be used by a plethora of other people. I know that the MySQL team had the habit to translate every error fixed into a test case in order to prevent that this error would creep in again. Well, eventually they had to to pay somebody full-time to run all these test cases. Sorry, my time is limited. 

Maybe I will change my mind if there is a program state which is stable in a sense, but so far I never reached this state, and I doubt I ever will.

My debug messages tell me everything I need and are inserted by PSPad shortcuts with a few keystrokes. 

Digression: Dirty example <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

For example, I produced a very weird problem somehow. Quite often, I load CodeIgniter modules in other CodeIgniter modules, which is no problem:

    $this->load->model('C_helper', 'ch'); 

The shortcut `ch` is optional. Next I can use this model with a simple syntax `$this->ch...`. As proof that this works, I might call a public property of this class with the shortcut `eex` and add `should be 5 DAY` in the title section and the call to the public property `$this->ch->check_interval` as a 2nd argument:

    xwp_echo("\nL: ".__LINE__."\n  :: \n  :: \nF: ".__FILE__."\nM: ".__METHOD__."\n" . wp_title(' should be 5 DAY '), $this->ch->check_interval );

This function `xwp_echo` is such a quick and dirty helper. The `x` can be added or deleted very quickly and will cause the whole process to stop if set and show the elapsed time. 

The function `wp_echo` is a simple `echo` with placeholders for easy addition of variable names to check their content plus information about the situation I am right in line to line number and method name and filename, but it takes a 2nd parameter, and if set will take care of proper formatting for strings, arrays and objects.

The output in this case is easy:

    L: 8425
      :: 
      :: 
    F: /www/application/models/Test.php
    M: Test::_load_model_M_helper
    12:44:55
    
     =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  => ||| should be 5 DAY
    
    seconds elapsed: 0.07 xwp_echo ============= L: 8425 ======= M: Test::_load_model_M_helper ::::::
    
    
    ---------------------------
    5 DAY 

So this works. How come I get the error

    A PHP Error was encountered
    
    Severity: Notice
    
    Message: Undefined property: Pages::$mh

    Line Number: 65
    
    Backtrace:
    
    File: /www/application/models/Ex_model.php
    Line: 3679
    Function: __get 

when calling the very similar

    xwp_echo("\nL: ".__LINE__."\n  :: \n  :: \nF: ".__FILE__."\nM: ".__METHOD__."\n" . wp_title(' should be 600 '), $this->mh->mem_lim );

I never had this kind of error, and I couldn't find anything about it via Google, except trivial faulty use. So no chance except finding myself. 

The first thing that comes to mind is that I use several classes here which obviously interfere. This shouldn't be a problem at all. And in fact it doesn't seem to be.

So I narrowed it down to the following scenario: for the function to work it needs an integer parameter. If this parameter is not set, it is 0 by default and nothing will happen. If not, I expect some nice output.

If I set the parameter before I load the module, I get the error. If I set it afterwards, everything is okay. Now this is hard to understand, isn't it? What happens here?

        $this->CI->id_ex = 6;         # This instruction triggers the error
    echo("<hr><pre> L: ".__LINE__."  ::  :: M: ".__METHOD__ . " F: ".__FILE__." ".date('H:i:s').' (  ) '."</pre>\n"   );
        $this->load->model('M_helper', 'mh'); 
    #xecho("<hr><pre> L: ".__LINE__."  ::  :: M: ".__METHOD__ . " F: ".__FILE__." ".date('H:i:s').' (  ) '."</pre>\n"   );
        $this->CI->id_ex = 6;        

Here you see 2 other nifty functions at work constructed very similar, but without the property of being able to display arrays and objects. The first one can be called before anything else is loaded, they are cheap, they are easy to read (just one line, as a rule) and with the `x` can be used to exit. Those functions stand out in the source code as all the debugging functions start in column one, so they are easy to detect. Example:   

    L: 789 key 108 id_avb 51519 M: Ex_common::_get_dm_entries 15:58:07 (  ) 
    
    L: 2841 id_avb :51519: this->id_ex :0: M: C_helper::_is_gs_error 15:58:07 ( NO check_now ) 
    
    L: 789 key 109 id_avb 49301 M: Ex_common::_get_dm_entries 15:58:07 (  ) 
    
    L: 2846 id_avb :49301: this->id_ex :0: M: C_helper::_is_gs_error 15:58:07 ( YES check_now ) 

Here you can see that I skim through a whole lot of entries of type `L: 789 key` to find the one where something will happen (`YES check_now`). 

Now back to the other problem. By commenting the first line `$this->CI->id_ex = 6;` above I can turn the error off, by uncommenting I turn it on. So what's happening here?

If you happen to load a module several times, that is no problem, CodeIgniter will handle that. The module technically is a property of the controller which is the base instance of all the modules. The loader class has an array which lists all the modules loaded.

This array is private, of course. Now my problem was that I had a call to a method of a class which was already loaded and I got an error saying that this module is not loaded. No, this is not correct. It says the module is no property of the controller class `Pages`. 

That's something different, but still when instantiated first, this module was set as a property to this class. And now the system tells me, that there is no such property. 

     Undefined property: Pages::$mh

How come? Who took it away? Or do I have different instances of that class? No, that cannot be.

In order to be able to check what the problem is here I wanted to see what this array in the loader class says. Maybe this entry had been erased by some enigmatic process.

I had already defined a subclass `WP_Loader` to the original `CI_Loader` class. That's the place to add a new method `get_ci_models`.

    // ====================================================================
    /**
     * get_ci_models()
     *
     * @access	public
     * @return	array
     */
    function get_ci_models() {
        return $this->_ci_models; 
    } # get_ci_models
    
Calling this function as a second argument to my debug function delivers the following state:

    L: 3672
      :: 
      :: 
    F: /www/application/models/Ex_model.php
    M: Ex_model::_get_ar_tn_tbl
    13:32:16
    
     =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  =>  => ||| This shows all the models loaded
    
    
    ---------------------------
    
        =================== :bgn array: ------------------: count(ar) = 5 :
    
        0  : ec
    
        1  : ex_model
    
        2  : test
    
        3  : ch
    
        4  : mh
    
    =================== :end array: ------------------ 

So this shows without doubt that the module `mh` I have trouble with is loaded. Confused.

So I have a model `test` which calls model `mh` which in turn calls model `ex_model` to invoke a method in `ex_model` which itself calls a method in `mh`. Now that's fairly complicated.

But the real circumstance that produces this trouble is the fact that I call this method of `ex_model` in the constructor of `mh` which is really dumb.          

I checked the existence of the models somewhere in the sequence, but when being in the constructor, this model cannot have been recorded yet. So `ex_model` in turn wants to call `mh` which, being constructed, isn't available yet. I should have asked for the existence of models at this place. Later on, this information doesn't tell me anything of use.

I don't pretend to have really understood what happened here, but this explanation sounds plausible. Anyway, removing my call in the constructor and placing it some other place totally resolved the whole issue.

For other very complex problems, I developed a technique where I could switch on or off this kind of dirty debug messages in certain functions via `GET` variables. This turned out to be very helpful as well.

This kind of debugging is really messy, I admit that. But I can comment any of these lines anytime in order to uncomment them whenever I should happen to need them again. That makes debugging very fast and easy. It's not elegant, it's not professional, but it simply works.

Digression: Adding microtime by trigger <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

That's all fine with that database approach, but still not really satisfactorily. The reason is that one second is too long a time to get the right sequence of commands. Hence the protocol is hard to read. And that's bad for debugging. 

Debugging basically means that you don't see what you should see. So the art of debugging is changing the conditions in order to be able to see what you should see. And if you see it, that's it: you can do something about it.  

As MySQL or MariaDB don't have a special data type `microtime`, I was looking for a solution via Google and first hit an attempt which is fine but nevertheless didn't convince me in the end: [Default value for “microtime” column in MySQL](https://dba.stackexchange.com/questions/114850/default-value-for-microtime-column-in-mysql).

    ALTER TABLE `tsmst`
    ADD `microtime` decimal(16,6) NOT NULL;
    
    DELIMITER //
    
    CREATE DEFINER=`root`@`localhost` TRIGGER `tmp`.`tsmst_BEFORE_INSERT` BEFORE INSERT ON `tsmst` FOR EACH ROW
     BEGIN
     SET NEW.microtime=UNIX_TIMESTAMP(NOW(6));
     END
     //
    
    DELIMITER ;

This is the first time that I defined a trigger. 

20 years ago, I had an employee who was a programming genius. I had to let him go and he ended up working for a company having a complex solution for big haulage contractors. 

He told me that they had huge problems with their customers because their database engine all of a sudden would do enigmatic operations and nobody would know what was happening. The reason was tons of triggers buried somewhere deep in the database nobody had an idea of, one calling the other under circumstances which were hard to test and debug.

That reminded me of programming rule number one: start simple and try to stay simple. It will become complex fast enough.

So from this reason alone I would object to introduce a trigger, although I admit that this solution looks very elegant. The second reason is that I had to change my PHP code. 

I had been lazy and did not list the columns when doing an insert, as I listed all the values anyway. Now my database had one more column, so these commands didn't work anymore. Okay, I changed everything in two PHP files, but then it turned out that I also had to change a shell script as well. At this point I hesitated.

Digression: Adding microtime natively <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Having a closer look, it turns out that the database engine can do microtime by itself now, so I could drop the microtime column and the trigger and nevertheless leave the corrected PHP code in place as a reminder that this can happen any time and it is good practice to list all columns for this reason.

    ALTER TABLE `tsmst`
    CHANGE `tmstmp` `tmstmp` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP AFTER `id_ex`,
    DROP `microtime`;

    DROP TRIGGER `tmp`.`tsmst_BEFORE_INSERT`;

Now I took the pain to clean up my code. I had inserted with copy and paste lots of `INSERT` statements, which I replaced by a function accepting one value for the column `comment`.

    function _tmp_tsmst_record($comment) {
        $comment = addSlashes($comment); 
        $sql = "INSERT INTO tmp.tsmst (id_ex, tmstmp, comment) VALUES ($this->id_ex, NOW(6), '$comment')";
        $query = $this->dba->query($sql . "\r\n# L: ".__LINE__.'. F:'.__FILE__.". M: ".__METHOD__);
    } # _tmp_tsmst_record

The term `NOW(6)` reflects the new definition of the timestamp column. And as I introduced this mechanism into several PHP files, I pushed this function definition high enough in the hierarchy to be used by all models. Well, the shell script also had to be corrected from `NOW()` to `NOW(6)`.

Now it looks really fine (in order to make it more readable for me and you I introduced empty lines):

    M:7727678 [tmp]>SELECT tmstmp, comment FROM tsmst WHERE id_ex = '2181' ORDER BY 1;
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    | tmstmp                     | comment                                                                                                     |
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    | 2018-03-09 19:09:45.077000 | tsmst.sh INIT en DEL :1:                                                                                    |
    
    | 2018-03-09 19:09:54.465791 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 nl 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.467829 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 en 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.469888 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 fr 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.471953 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 zh 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.473298 | 24663 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 de 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    
    | 2018-03-09 19:10:02.085619 | tsmst.sh INIT nl DEL :0:                                                                                    |
    | 2018-03-09 19:10:02.144768 | tsmst.sh INIT en DEL :0:                                                                                    |
    | 2018-03-09 19:10:02.214229 | tsmst.sh INIT fr DEL :0:                                                                                    |
    | 2018-03-09 19:10:02.275222 | tsmst.sh INIT zh DEL :0:                                                                                    |
    
    | 2018-03-09 19:13:18.152125 | 22384 while fr done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:18.201954 | 24808 DROP TABLE IF EXISTS bak.tn_2181_fr                                                                   |
    | 2018-03-09 19:13:18.249423 | 24819 INSERT INTO bak.tn_2181_fr SELECT * FROM tmp.tn_2181_fr                                               |
    | 2018-03-09 19:13:18.345219 | 24765 done INSERT INTO tn_fr                                                                                |
    | 2018-03-09 19:13:18.349045 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:18.573786 | == GOOD!!!===== LG :fr: === used :197: secs                                                                 |
    
    | 2018-03-09 19:13:28.585231 | 22384 while en done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:28.593704 | 24808 DROP TABLE IF EXISTS bak.tn_2181_en                                                                   |
    | 2018-03-09 19:13:28.623794 | 24819 INSERT INTO bak.tn_2181_en SELECT * FROM tmp.tn_2181_en                                               |
    | 2018-03-09 19:13:29.399184 | 24765 done INSERT INTO tn_en                                                                                |
    | 2018-03-09 19:13:29.401784 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:29.568019 | == GOOD!!!===== LG :en: === used :208: secs                                                                 |
    
    | 2018-03-09 19:13:34.380006 | 22384 while zh done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:34.392166 | 24808 DROP TABLE IF EXISTS bak.tn_2181_zh                                                                   |
    | 2018-03-09 19:13:34.453988 | 24819 INSERT INTO bak.tn_2181_zh SELECT * FROM tmp.tn_2181_zh                                               |
    | 2018-03-09 19:13:34.641064 | 24765 done INSERT INTO tn_zh                                                                                |
    | 2018-03-09 19:13:34.644690 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:34.835122 | == GOOD!!!===== LG :zh: === used :213: secs                                                                 |
    
    | 2018-03-09 19:13:46.975934 | 22384 while nl done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:46.985674 | 24808 DROP TABLE IF EXISTS bak.tn_2181_nl                                                                   |
    | 2018-03-09 19:13:47.010678 | 24819 INSERT INTO bak.tn_2181_nl SELECT * FROM tmp.tn_2181_nl                                               |
    | 2018-03-09 19:13:47.367528 | 24765 done INSERT INTO tn_nl                                                                                |
    | 2018-03-09 19:13:47.371394 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:47.560558 | == GOOD!!!===== LG :nl: === used :226: secs                                                                 |
    
    | 2018-03-09 19:15:58.444864 | 22384 while de done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:15:58.452480 | 24808 DROP TABLE IF EXISTS bak.tn_2181_de                                                                   |
    | 2018-03-09 19:15:58.478059 | 24819 INSERT INTO bak.tn_2181_de SELECT * FROM tmp.tn_2181_de                                               |
    | 2018-03-09 19:15:58.630610 | 24765 done INSERT INTO tn_de         
                                                              |
    | 2018-03-09 19:15:58.634881 | 24844 nl 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.637278 | 2455 lg :nl: CI->lg :nl: 2181  Ex_model::_build_tn                                                          |
    | 2018-03-09 19:15:58.640836 | 2575 nl :nl: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.644141 | 2638 nl :nl: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.649960 | 3109 nl :nl: 2181 Ex_model::_get_tn_lg_links                                                                |
    | 2018-03-09 19:15:58.650381 | 3113 this->lg :nl: here we set em5 Ex_model::_get_tn_lg_links                                               |
    | 2018-03-09 19:15:58.715690 | 1452 this->lg :nl: token :9d4ee0c51b36a4a5c4de82f2a0449b69: cmp_temp md5 XQ_Model::_cmp_lg_set              |
    | 2018-03-09 19:15:58.722742 | 2703 nl :nl: 2181 before _cmp_lg_set Ex_model::_build_tn                                                    |
    | 2018-03-09 19:15:58.726384 | 1452 this->lg :nl: token :2181: cmp_ex_tn id_ex XQ_Model::_cmp_lg_set                                       |
    
    | 2018-03-09 19:15:58.726843 | 24844 en 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.727288 | 2455 lg :en: CI->lg :en: 2181  Ex_model::_build_tn                                                          |
    | 2018-03-09 19:15:58.738060 | 2575 en :en: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.741976 | 2638 en :en: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.748151 | 3109 en :en: 2181 Ex_model::_get_tn_lg_links                                                                |
    | 2018-03-09 19:15:58.748594 | 3113 this->lg :en: here we set em5 Ex_model::_get_tn_lg_links                                               |
    | 2018-03-09 19:15:58.796012 | 1452 this->lg :en: token :219c3878ac7ae59d9341e52246047eaf: cmp_temp md5 XQ_Model::_cmp_lg_set              |
    | 2018-03-09 19:15:58.800604 | 2703 en :en: 2181 before _cmp_lg_set Ex_model::_build_tn                                                    |
    | 2018-03-09 19:15:58.804053 | 1452 this->lg :en: token :2181: cmp_ex_tn id_ex XQ_Model::_cmp_lg_set                                       |
    
    | 2018-03-09 19:15:58.804513 | 24844 fr 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.804955 | 2455 lg :fr: CI->lg :fr: 2181  Ex_model::_build_tn                                                          |
    | 2018-03-09 19:15:58.815546 | 2575 fr :fr: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.820236 | 2638 fr :fr: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.827271 | 3109 fr :fr: 2181 Ex_model::_get_tn_lg_links                                                                |
    | 2018-03-09 19:15:58.827680 | 3113 this->lg :fr: here we set em5 Ex_model::_get_tn_lg_links                                               |
    | 2018-03-09 19:15:58.855871 | 1452 this->lg :fr: token :302547b53945d521b9124c2855a5f91a: cmp_temp md5 XQ_Model::_cmp_lg_set              |
    | 2018-03-09 19:15:58.859158 | 2703 fr :fr: 2181 before _cmp_lg_set Ex_model::_build_tn                                                    |
    | 2018-03-09 19:15:58.862583 | 1452 this->lg :fr: token :2181: cmp_ex_tn id_ex XQ_Model::_cmp_lg_set                                       |
    
    | 2018-03-09 19:15:58.863054 | 24844 zh 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.863506 | 2455 lg :zh: CI->lg :zh: 2181  Ex_model::_build_tn                                                          |
    | 2018-03-09 19:15:58.866537 | 2575 zh :zh: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.870033 | 2638 zh :zh: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.876537 | 3109 zh :zh: 2181 Ex_model::_get_tn_lg_links                                                                |
    | 2018-03-09 19:15:58.877004 | 3113 this->lg :zh: here we set em5 Ex_model::_get_tn_lg_links                                               |
    | 2018-03-09 19:15:58.897932 | 1452 this->lg :zh: token :8b431ae02edcbdf2afc6528c5688a963: cmp_temp md5 XQ_Model::_cmp_lg_set              |
    | 2018-03-09 19:15:58.901354 | 2703 zh :zh: 2181 before _cmp_lg_set Ex_model::_build_tn                                                    |
    | 2018-03-09 19:15:58.904851 | 1452 this->lg :zh: token :2181: cmp_ex_tn id_ex XQ_Model::_cmp_lg_set                                       |
    
    | 2018-03-09 19:15:58.905317 | 24844 de 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.905766 | 2455 lg :de: CI->lg :de: 2181  Ex_model::_build_tn                                                          |
    | 2018-03-09 19:15:58.919954 | 2575 de :de: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.923323 | 2638 de :de: 2181 Ex_model::_build_tn                                                                       |
    | 2018-03-09 19:15:58.931337 | 3109 de :de: 2181 Ex_model::_get_tn_lg_links                                                                |
    | 2018-03-09 19:15:58.931844 | 3113 this->lg :de: here we set em5 Ex_model::_get_tn_lg_links                                               |
    | 2018-03-09 19:15:58.955611 | 1452 this->lg :de: token :89f1427a4efa3002069afe6c5019f27f: cmp_temp md5 XQ_Model::_cmp_lg_set              |
    | 2018-03-09 19:15:58.959340 | 2703 de :de: 2181 before _cmp_lg_set Ex_model::_build_tn                                                    |
    | 2018-03-09 19:15:58.963374 | 1452 this->lg :de: token :2181: cmp_ex_tn id_ex XQ_Model::_cmp_lg_set                                       |
    | 2018-03-09 19:15:58.964101 | 24850 de 2181  this->_current_tn_lg :en:                                                                    |
    | 2018-03-09 19:15:59.173544 | == GOOD!!!===== LG :en: === used :375: secs                                                                 |
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    85 rows in set (0.00 sec)

Here `too early` means that I have to wait for all language processes to be completed before I can sum up with `Ex_model->_build_tn`. 

Some lines seem to be redundant -- you see that I track the value of a variable which seemed to be wrong. The problem was not where I thought it would be. However, I found out where it was with the same technique and fixed the problem, so what you see here is just the remainder of debug messages which is clean now. 

Omitting these lines which are now superfluous, the whole picture is even clearer:

    M:7727678 [tmp]>SELECT tmstmp, comment FROM tsmst WHERE id_ex = '2181' ORDER BY 1;
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    | tmstmp                     | comment                                                                                                     |
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    | 2018-03-09 19:09:45.077000 | tsmst.sh INIT en DEL :1:                                                                                    |
    
    | 2018-03-09 19:09:54.465791 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 nl 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.467829 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 en 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.469888 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 fr 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.471953 | 24670 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 zh 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-09 19:09:54.473298 | 24663 _tsmst_write_trigger nohup /c/bak/tsmst.sh 2181 de 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    
    | 2018-03-09 19:10:02.085619 | tsmst.sh INIT nl DEL :0:                                                                                    |
    | 2018-03-09 19:10:02.144768 | tsmst.sh INIT en DEL :0:                                                                                    |
    | 2018-03-09 19:10:02.214229 | tsmst.sh INIT fr DEL :0:                                                                                    |
    | 2018-03-09 19:10:02.275222 | tsmst.sh INIT zh DEL :0:                                                                                    |
    
    | 2018-03-09 19:13:18.152125 | 22384 while fr done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:18.201954 | 24808 DROP TABLE IF EXISTS bak.tn_2181_fr                                                                   |
    | 2018-03-09 19:13:18.249423 | 24819 INSERT INTO bak.tn_2181_fr SELECT * FROM tmp.tn_2181_fr                                               |
    | 2018-03-09 19:13:18.345219 | 24765 done INSERT INTO tn_fr                                                                                |
    | 2018-03-09 19:13:18.349045 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:18.573786 | == GOOD!!!===== LG :fr: === used :197: secs                                                                 |
    
    | 2018-03-09 19:13:28.585231 | 22384 while en done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:28.593704 | 24808 DROP TABLE IF EXISTS bak.tn_2181_en                                                                   |
    | 2018-03-09 19:13:28.623794 | 24819 INSERT INTO bak.tn_2181_en SELECT * FROM tmp.tn_2181_en                                               |
    | 2018-03-09 19:13:29.399184 | 24765 done INSERT INTO tn_en                                                                                |
    | 2018-03-09 19:13:29.401784 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:29.568019 | == GOOD!!!===== LG :en: === used :208: secs                                                                 |
    
    | 2018-03-09 19:13:34.380006 | 22384 while zh done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:34.392166 | 24808 DROP TABLE IF EXISTS bak.tn_2181_zh                                                                   |
    | 2018-03-09 19:13:34.453988 | 24819 INSERT INTO bak.tn_2181_zh SELECT * FROM tmp.tn_2181_zh                                               |
    | 2018-03-09 19:13:34.641064 | 24765 done INSERT INTO tn_zh                                                                                |
    | 2018-03-09 19:13:34.644690 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:34.835122 | == GOOD!!!===== LG :zh: === used :213: secs                                                                 |
    
    | 2018-03-09 19:13:46.975934 | 22384 while nl done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:13:46.985674 | 24808 DROP TABLE IF EXISTS bak.tn_2181_nl                                                                   |
    | 2018-03-09 19:13:47.010678 | 24819 INSERT INTO bak.tn_2181_nl SELECT * FROM tmp.tn_2181_nl                                               |
    | 2018-03-09 19:13:47.367528 | 24765 done INSERT INTO tn_nl                                                                                |
    | 2018-03-09 19:13:47.371394 | 24838 de 2181 _build_tn too early                                                                           |
    | 2018-03-09 19:13:47.560558 | == GOOD!!!===== LG :nl: === used :226: secs                                                                 |
    
    | 2018-03-09 19:15:58.444864 | 22384 while de done, about to _transfer_tmp_to_dj5                                                          |
    | 2018-03-09 19:15:58.452480 | 24808 DROP TABLE IF EXISTS bak.tn_2181_de                                                                   |
    | 2018-03-09 19:15:58.478059 | 24819 INSERT INTO bak.tn_2181_de SELECT * FROM tmp.tn_2181_de                                               |
    | 2018-03-09 19:15:58.630610 | 24765 done INSERT INTO tn_de         
                                                              |
    | 2018-03-09 19:15:58.634881 | 24844 nl 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.726843 | 24844 en 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.804513 | 24844 fr 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.863054 | 24844 zh 2181 trigger Ex_model->_build_tn -----------------                                                 |
    | 2018-03-09 19:15:58.905317 | 24844 de 2181 trigger Ex_model->_build_tn -----------------                                                 |

    | 2018-03-09 19:15:58.964101 | 24850 de 2181  this->_current_tn_lg :en:                                                                    |
    | 2018-03-09 19:15:59.173544 | == GOOD!!!===== LG :en: === used :375: secs                                                                 |
    +----------------------------+-------------------------------------------------------------------------------------------------------------+

Working more with this approach, I found that `varchar (255)` is not enough:

    ALTER TABLE `tsmst`
    CHANGE `comment` `comment` longtext COLLATE 'utf8mb4_unicode_ci' NOT NULL AFTER `tmstmp`;

The picture is not that clear when more than one language needs exactly the same time to complete its job. Then the different actions intertwine again:

Example:

    | 2018-03-12 13:48:28.721303 | 24875 _db_copy_tmp_to_bak DROP TABLE IF EXISTS bak.tn_1624_en                         |
    | 2018-03-12 13:48:28.778219 | 24886 _db_copy_tmp_to_bak INSERT INTO bak.tn_1624_en SELECT * FROM tmp.tn_1624_en     |
    | 2018-03-12 13:48:28.824289 | 24832 _transfer_tmp_to_dj5 done INSERT INTO tn_en, try to _build_tns                  |
    | 2018-03-12 13:48:28.827740 | 24912 _build_tns en -- de not ready yet 1624 _build_tn too early                      |
    | 2018-03-12 13:48:29.277281 | 24875 _db_copy_tmp_to_bak DROP TABLE IF EXISTS bak.tn_1624_it                         |
    | 2018-03-12 13:48:29.375357 | 24886 _db_copy_tmp_to_bak INSERT INTO bak.tn_1624_it SELECT * FROM tmp.tn_1624_it     |
    | 2018-03-12 13:48:29.407542 | 24832 _transfer_tmp_to_dj5 done INSERT INTO tn_it, try to _build_tns                  |
    | 2018-03-12 13:48:29.410364 | 24912 _build_tns it -- de not ready yet 1624 _build_tn too early                      |
    | 2018-03-12 13:48:29.493763 | == GOOD!!!===== LG :en: === used :28: secs                                            |
    | 2018-03-12 13:48:29.818472 | 24875 _db_copy_tmp_to_bak DROP TABLE IF EXISTS bak.tn_1624_es                         |
    | 2018-03-12 13:48:29.845114 | 24886 _db_copy_tmp_to_bak INSERT INTO bak.tn_1624_es SELECT * FROM tmp.tn_1624_es     |
    | 2018-03-12 13:48:29.885764 | 24832 _transfer_tmp_to_dj5 done INSERT INTO tn_es, try to _build_tns                  |
    | 2018-03-12 13:48:29.890009 | 24912 _build_tns es -- de not ready yet 1624 _build_tn too early                      |
    | 2018-03-12 13:48:29.997682 | 24875 _db_copy_tmp_to_bak DROP TABLE IF EXISTS bak.tn_1624_ru                         |
    | 2018-03-12 13:48:30.059024 | 24886 _db_copy_tmp_to_bak INSERT INTO bak.tn_1624_ru SELECT * FROM tmp.tn_1624_ru     |
    | 2018-03-12 13:48:30.094415 | 24832 _transfer_tmp_to_dj5 done INSERT INTO tn_ru, try to _build_tns                  |
    | 2018-03-12 13:48:30.098471 | 24912 _build_tns ru -- de not ready yet 1624 _build_tn too early                      |
    | 2018-03-12 13:48:30.248740 | == GOOD!!!===== LG :it: === used :29: secs                                            |
    | 2018-03-12 13:48:30.563944 | == GOOD!!!===== LG :es: === used :29: secs                                            |
    | 2018-03-12 13:48:30.744574 | 24875 _db_copy_tmp_to_bak DROP TABLE IF EXISTS bak.tn_1624_fr                         |
    | 2018-03-12 13:48:30.767134 | 24886 _db_copy_tmp_to_bak INSERT INTO bak.tn_1624_fr SELECT * FROM tmp.tn_1624_fr     |
    | 2018-03-12 13:48:30.778874 | 24832 _transfer_tmp_to_dj5 done INSERT INTO tn_fr, try to _build_tns                  |
    | 2018-03-12 13:48:30.782254 | 24912 _build_tns fr -- de not ready yet 1624 _build_tn too early                      |
    | 2018-03-12 13:48:30.825544 | == GOOD!!!===== LG :ru: === used :29: secs                                            |
    | 2018-03-12 13:48:31.437410 | == GOOD!!!===== LG :fr: === used :30: secs                                            |
    
I guess the best way to get a clearer picture will then be to filter the result by adding conditions to the query. The conditions are not that easy with short snippets like `fr` which are part of something else, in this case even the SQL syntax `FROM`, so there is more hassle here, but it can be done:

    M:7727678 [tmp]>SELECT tmstmp, comment FROM tsmst 
    	WHERE id_ex = '1624' 
    	AND (
            comment LIKE '% fr %' 
            OR 
            comment LIKE '%\_fr%' 
            OR 
            comment LIKE '%:fr:%'
            ) 
    	ORDER BY 1;
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    | tmstmp                     | comment                                                                                                     |
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    | 2018-03-12 13:55:03.321134 | 24743 _tsmst_write_trigger nohup /c/bak/tsmst.sh 1624 fr 0 2>&1 1>>/tmp/good.tsms3 0</dev/null 1>&/dev/null |
    | 2018-03-12 13:56:01.976050 | tsmst.sh INIT fr DEL :0:                                                                                    |
    | 2018-03-12 13:56:02.743360 | 24912 _build_tns fr -- de not ready yet 1624 _build_tn too early                                            |
    | 2018-03-12 13:56:33.790927 | 24875 _db_copy_tmp_to_bak DROP TABLE IF EXISTS bak.tn_1624_fr                                               |
    | 2018-03-12 13:56:33.820771 | 24886 _db_copy_tmp_to_bak INSERT INTO bak.tn_1624_fr SELECT * FROM tmp.tn_1624_fr                           |
    | 2018-03-12 13:56:33.843314 | 24832 _transfer_tmp_to_dj5 done INSERT INTO tn_fr, try to _build_tns                                        |
    | 2018-03-12 13:56:33.861680 | 24912 _build_tns fr -- de not ready yet 1624 _build_tn too early                                            |
    | 2018-03-12 13:56:34.894510 | == GOOD!!!===== LG :fr: === used :33: secs                                                                  |
    | 2018-03-12 13:58:51.315580 | 24924 _build_tns fr Versions linguistiques                                                                  |
    +----------------------------+-------------------------------------------------------------------------------------------------------------+
    8 rows in set (0.00 sec)

Digression: Language versions <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

One more example with another ID which uses the languages German, English, Spanish, French, Italian and Russian. Using this database technique, I easily found the problem of not getting the right translation for a specific word when I was wrapping up all the language versions after having collected all data separately. 

    | 2018-03-10 14:10:03.617272 | 24774 _transfer_tmp_to_dj5 done INSERT INTO tn_de, try to _build_tns |

Here everything is done and we try to build the result for de, en, es, fr, it, ru:

    | 2018-03-10 14:10:03.630156 | 24855 _build_tns de 1624 trigger Ex_model->_build_tn --------------- |
    | 2018-03-10 14:10:03.665357 | 3173 Ex_model::_get_tn_lg_links this->lg :de: Sprachversionen        |
    | 2018-03-10 14:10:03.696897 | 24855 _build_tns en 1624 trigger Ex_model->_build_tn --------------- |
    | 2018-03-10 14:10:03.723393 | 3173 Ex_model::_get_tn_lg_links this->lg :en: Language versions      |
    | 2018-03-10 14:10:03.753933 | 24855 _build_tns es 1624 trigger Ex_model->_build_tn --------------- |
    | 2018-03-10 14:10:03.773810 | 3173 Ex_model::_get_tn_lg_links this->lg :es: Versiones de idioma    |
    | 2018-03-10 14:10:03.803189 | 24855 _build_tns fr 1624 trigger Ex_model->_build_tn --------------- |
    | 2018-03-10 14:10:03.830543 | 3173 Ex_model::_get_tn_lg_links this->lg :fr: Versions linguistiques |
    | 2018-03-10 14:10:03.861603 | 24855 _build_tns it 1624 trigger Ex_model->_build_tn --------------- |
    | 2018-03-10 14:10:03.881396 | 3173 Ex_model::_get_tn_lg_links this->lg :it: Versioni linguistiche  |
    | 2018-03-10 14:10:03.910080 | 24855 _build_tns ru 1624 trigger Ex_model->_build_tn --------------- |
    | 2018-03-10 14:10:03.930972 | 3173 Ex_model::_get_tn_lg_links this->lg :ru: Языковые               |

We switch back to the original language we began our operations with:

    | 2018-03-10 14:10:03.957584 | 24861 _build_tns ru 1624  this->_current_tn_lg :en:                  |
    | 2018-03-10 14:10:04.145263 | == GOOD!!!===== LG :en: === used :259: secs                          |

The result shows that one by one those 6 languages are switched and the correct translation is found: 

- Sprachversionen, 
- Language versions, 
- Versiones de idioma, 
- Versions linguistiques, 
- Versioni linguistiche, 
- Языковые.

The same for the former example:

    | 2018-03-10 14:44:50.655962 | 3173 Ex_model::_get_tn_lg_links this->lg :nl: Taalversies            |
    | 2018-03-10 14:44:50.698879 | 3173 Ex_model::_get_tn_lg_links this->lg :en: Language versions      |
    | 2018-03-10 14:44:50.804486 | 3173 Ex_model::_get_tn_lg_links this->lg :fr: Versions linguistiques |
    | 2018-03-10 14:44:50.840104 | 3173 Ex_model::_get_tn_lg_links this->lg :zh: 语言版本               |
    | 2018-03-10 14:44:50.888654 | 3173 Ex_model::_get_tn_lg_links this->lg :de: Sprachversionen        |

Digression: Analyzing data <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

There is so much you can do with databases:

    M:7727678 [tmp]>SELECT id_ex, tmstmp, comment,
        -> MID(comment, 21, 2) lg,
        -> REPLACE(SUBSTRING(comment, 35), ':', '') time
        -> FROM tsmst
        -> WHERE 1
        -> AND id_ex IN (1624, 2181, 391)
        -> AND comment LIKE '%good!%'
        -> ORDER BY 1, 2;
    +-------+----------------------------+----------------------------------------------+----+-----------+
    | id_ex | tmstmp                     | comment                                      | lg | time      |
    +-------+----------------------------+----------------------------------------------+----+-----------+
    |   391 | 2018-03-10 21:27:50.293078 | == GOOD!!!===== LG :de: === used :109: secs  | de | 109 secs  |
    |   391 | 2018-03-10 21:27:51.615886 | == GOOD!!!===== LG :en: === used :110: secs  | en | 110 secs  |
    |   391 | 2018-03-10 21:27:54.237891 | == GOOD!!!===== LG :nl: === used :113: secs  | nl | 113 secs  |
    |   391 | 2018-03-10 21:28:20.194331 | == GOOD!!!===== LG :fr: === used :138: secs  | fr | 138 secs  |
    |  1624 | 2018-03-10 16:21:51.781247 | == GOOD!!!===== LG :fr: === used :49: secs   | fr | 49 secs   |
    |  1624 | 2018-03-10 16:21:51.962183 | == GOOD!!!===== LG :es: === used :49: secs   | es | 49 secs   |
    |  1624 | 2018-03-10 16:21:52.020165 | == GOOD!!!===== LG :it: === used :49: secs   | it | 49 secs   |
    |  1624 | 2018-03-10 16:21:52.098927 | == GOOD!!!===== LG :en: === used :50: secs   | en | 50 secs   |
    |  1624 | 2018-03-10 16:21:52.205438 | == GOOD!!!===== LG :ru: === used :50: secs   | ru | 50 secs   |
    |  1624 | 2018-03-10 16:26:41.156284 | == GOOD!!!===== LG :en: === used :361: secs  | en | 361 secs  |
    |  2181 | 2018-03-10 15:28:59.089538 | == GOOD!!!===== LG :fr: === used :177: secs  | fr | 177 secs  |
    |  2181 | 2018-03-10 15:29:20.260428 | == GOOD!!!===== LG :zh: === used :199: secs  | zh | 199 secs  |
    |  2181 | 2018-03-10 15:29:21.306236 | == GOOD!!!===== LG :en: === used :200: secs  | en | 200 secs  |
    |  2181 | 2018-03-10 15:29:33.003864 | == GOOD!!!===== LG :nl: === used :211: secs  | nl | 211 secs  |
    |  2181 | 2018-03-10 15:31:14.793069 | == GOOD!!!===== LG :en: === used :367: secs  | en | 367 secs  |
    +-------+----------------------------+----------------------------------------------+----+-----------+
    14 rows in set (0.00 sec)

It's true, you can do much with `grep` and `awk` and `sed` as well, but it doesn't come as naturally and much isn't possible at all. There's a reason why relational databases exist, and if you have one at your disposal, you will most probably take advantage of it.

Looking at these numbers, it is obvious that there are 2 lines which stand out by the `time` value:

    |  1624 | 2018-03-10 16:26:41.156284 | == GOOD!!!===== LG :en: === used :361: secs  | en | 361 secs  |
    |  2181 | 2018-03-10 15:31:14.793069 | == GOOD!!!===== LG :en: === used :367: secs  | en | 367 secs  |

Also, `en` appears twice in both ID cases 1624 and 2181. You may not have noticed, but the message of the shell file is wrong, it shows the wrong language, which is clear from the context:

    | 2018-03-10 16:26:40.629453 | 24774 _transfer_tmp_to_dj5 done INSERT INTO tn_de, try to _build_tns |
    | 2018-03-10 16:26:40.641469 | 24855 _build_tns de 1624 trigger Ex_model->_build_tn --------------- |

    | 2018-03-10 15:31:14.327917 | 24774 _transfer_tmp_to_dj5 done INSERT INTO tn_de, try to _build_tns |
    | 2018-03-10 15:31:14.336398 | 24855 _build_tns de 1624 trigger Ex_model->_build_tn --------------- |

So the mechanism is as follows: we start with `en`, and this is what the shell script (`== GOOD!!!`) knows. 

For some reason, in these cases the language `en` is then switched to `de` to be processed first, which the shell script does not know of. Maybe I could invent a mechanism to inform the shell about this transition, but I don't think it's worth the effort. Anyway, this explains why the value `en` in this line is wrong and should be `de` instead. 

For another reason `de` is the last one to be completed in both cases, so only then the windup can start. The time it takes to process all the languages one by one adds up to the total time this last language needs. In these both cases the time span used was significant so the mismatch leaps to the eye.

The average time taken for processing these languages is obviously very different:

    |   ID | number of lg | avg. time first run per lg | avg. time windup per lg |
    |  391 |            4 |                         49 |                       7 |
    | 1624 |            6 |                        110 |                      52 |
    | 2181 |            5 |                        196 |                      34 |

The value for `average time first run` is calculated on the basis of all but the last runs. This value and the `average time windup` do not depend on the number of languages but on the nature of the data to be processed.

Digression: Adding a stopwatch by database <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Now I was bitten by the bug and realize that I need more. That whole time taking by the shell isn't what I need. I want to start a whole lot of shell scripts and have an easy way to monitor these processes. Basically I'm only interested in those running and the time each one takes if it finishes. So I need a new table:

 CREATE TABLE `tsmst_time` (
  `id_ex` bigint(20) unsigned NOT NULL,
  `tmstmp` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `comment` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  PRIMARY KEY (`id_ex`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

For a moment I was irritated because I didn't want to timestamp column to be updated automatically and couldn't manage to change this definition, so I thought about it and remembered that the first `timestamp` value is automatically, no matter what, updated -- which is a good thing. So in order to preserve the original `timestamp` when the record was created I could introduce a second `timestamp` column, but as I am only interested in the difference anyway and will record this difference in the `comment` column, that's all right.

Next I introduced a new function:

    // ====================================================================
    /**
     * _tmp_tsmst_record($comment)
     *
     * @access	public
     * @param 	string
     * @return	void
     */
    function _tmp_tsmst_time_record($comment) {
        $sql = "UPDATE tmp.tsmst_time set comment = '$comment' WHERE id_ex = '$this->id_ex'";
        $query = $this->dba->query($sql . "\r\n# L: ".__LINE__.'. F:'.__FILE__.". M: ".__METHOD__);
    } # _tmp_tsmst_record

This function will take care of recoding the time taken. The beginning is recorded in the constructor of my class; to this end, I introduced a new private member `$_microtime`:

        $this->_microtime = microtime(true); # this is the beginning of time taking
        $this->_tmp_tsmst_record(__LINE__ . ' ' . __METHOD__ . " $this->lg $this->id_ex  ");
        # record the beginning of the whole process in our monitoring table as well
        $sql = "REPLACE INTO tmp.tsmst_time (id_ex, tmstmp, comment) VALUES ($this->id_ex, NOW(6), '')";
        $query = $this->dba->query($sql . "\r\n# L: ".__LINE__.'. F:'.__FILE__.". M: ".__METHOD__);
        # make sure that we only have one record for each $this->id_ex

So when a process starts, we will know because there is one row in this time taking table, and as long as the process runs, the value for `comment` will be empty. 

    M:7727678 [tmp]>select * from tsmst_time ORDER BY 1;
    +-------+----------------------------+-----------------+
    | id_ex | tmstmp                     | comment         |
    +-------+----------------------------+-----------------+
    |     6 | 2018-03-15 18:34:10.463571 | 17.235496044159 |
    |  1624 | 2018-03-15 18:33:58.407854 |                 |
    |  2181 | 2018-03-15 18:34:03.484912 |                 |
    +-------+----------------------------+-----------------+

When the process finishes, we take the time and update this row. We not only know that the process has finished, but also how much time it has taken.

        $time_used = microtime(true) - $this->_microtime; 
        $this->_tmp_tsmst_time_record($time_used);

To round up things, I additionally write this value to the monitoring table as well:

        $this->_tmp_tsmst_record(__LINE__ . ' ' . __METHOD__ . " done $lg $this->id_ex :$this->_current_sitemap_lg: exhibitor_model->_show_ex_sitemap time_used :$time_used:");

Interestingly, the whole process takes much more time when started from the shell versus the browser. I have no idea why this is so. The browser lives on a different machine and has to transmit its message across the network. I would have thought that the relation would've been just the opposite. For example, instead of these 17 seconds recorded in this sample the values taken from the browser are 12 or 13 seconds, which will not be significant if the whole process takes much longer.

Soon I found out that I didn't think far enough. The example I was testing this enhancement with was the simplest I could get, so it only works in one language. But the next one had a couple more, and then I found that each language would call the constructor, killing my entry. So I had to introduce the language as well and also change the primary key in order to make the construct `REPLACE INTO` work as expected.

    ALTER TABLE `tsmst_time`
    ADD `lg` varchar(2) NOT NULL AFTER `id_ex`,
    CHANGE `comment` `comment` varchar(25) COLLATE 'utf8mb4_unicode_ci' NOT NULL AFTER `tmstmp`;
    
    ALTER TABLE `tsmst_time`
    ADD PRIMARY KEY `id_ex_lg` (`id_ex`, `lg`),
    DROP INDEX `PRIMARY`;

Of course, the PHP code had to be updated as well

    // ====================================================================
    /**
     * _tmp_tsmst_record($comment)
     *
     * @access	public
     * @param 	string
     * @return	void
     */
    function _tmp_tsmst_time_record($comment) {
        $comment = addSlashes($comment); 
        $sql = "UPDATE tmp.tsmst_time set comment = '$comment' WHERE id_ex = '$this->id_ex' AND lg = '$this->lg'";
        $query = $this->dba->query($sql . "\r\n# L: ".__LINE__.'. F:'.__FILE__.". M: ".__METHOD__);
    } # _tmp_tsmst_record


        $sql = "REPLACE INTO tmp.tsmst_time (id_ex, lg, tmstmp, comment) VALUES ('$this->id_ex', '$this->lg', NOW(6), '')";

You may notice that I put all my values in `'` even when not necessary. Actually I didn't here, if you look back, which was okay here because `$this->id_ex` is an integer, and because of that, when inserting the new value `$this->lg`, I wasn't aware that this is a string and must be enclosed by `'`. Well, it didn't take long until I found out because my program didn't work anymore.

The whole investigation presented here is not just for fun or educational purposes. I have rearranged central parts of my code and refactored a major mechanism for simplification and empowerment which usually is not easy and prone to introduce lots of new bugs. This technique has saved me much time and effort. 

I'm glad I have developed it. I'm not sure if this would have happened if I wouldn't have taken the pain to describe what I did in this article -- well, it developed into a kind of a diary. It was interesting for me, at least.

Digression: Erlang style <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

You may wonder about the big line numbers in one of those PHP files. That's not a problem but due to the complexity of the task being addressed by this class. A problem are the long function definitions. There are quite a lot of methods in that class defined in that file, but still many of these functions are really really big. And that's not really good.

I learned that from my first experiences with `Erlang`. Functions in Erlang are short, sometimes extremely short. Nevertheless they produce programs in Erlang with millions of rows. 

It was quite an experience to get a grip on Erlang in order to be able to produce productive code. I wish I had learned something along these lines earlier so that I could have used that experience in my programming habits in PHP as well.

Of course, the problem of debugging that Erlang code arose as well. Guess what, I wrote my own debugging functions in Erlang in order to enhance my productivity.

Well, looking at the source code of CodeIgniter, there are lots of functions which are extremely short. I could have learned from them as well, but alas I didn't. Frankly, I didn't study their source code if I didn't have to -- I was too eager to become productive. To be fair, they do have plenty of very long functions as well. 

Another problem probably is that I don't have any peer review anymore for about 20 years now. Also it is difficult to explore new realms producing almost inevitably quick and dirty code and then afterwards getting clean and lean code easy to maintain. When does development stop? When do you have time to optimize your code?

Often I think a good program will have at least 3 stages: 

- development first, when you don't know yet exactly what you want or should be after, 

- next you most probably know what you're heading at so you can rewrite the whole thing from scratch, 

- and after that you may understand what the whole endeavor is about so you can rewrite once again. 

And maybe even that is not the last iteration. This expensive scenario reminds me very much of what some authors tell about their way of producing novels. 

Well, not every author works this way, some just write down what comes to mind and it is brilliant nevertheless. Maybe there are programmers out there who do work like that.

End of digression.

Search engines <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

This text will be indexed by search engines and may be found for quite some time to come by people like me looking for a solution to their problems related to any of the search-relevant technical terms I have used. 

As people in times of Docker containers tend to use minimal Linux systems like `Boot2Docker`, `CoreOS` or `Alpine Linux` using `ash` instead of `bash`, most probably there will be more need for a substitute for LINENO. That's the path I have taken. Those 2 MySQL (or rather MariaDB) replication engines run in Docker containers as does the master. The OS is `Boot2Docker` which is based on `Tiny Linux` which in turn is based on `Busybox`. No `bash`, only `ash`.

Furthermore, I hope that you, having read so far, did enjoy the article and don't regret having spent your time.

A big thank you to you all <span style="font-size: 11px;float: right;"><a href="#toc">Table of Content</a></span>
----------

Last but not least I want to thank all you experts out there emphatically for all your excellent work. There are very many fine solutions I've found for my problems which have been incorporated in my setup without a note.

Nowadays, whenever I have a problem without having an idea, I immediately call Google, and if there is a solution, most probably I find it this way. If nobody had this kind of problem, there is a good chance that the problem isn't where I think it is.

It's hard to look into the future, but we can look back. My first encounter with computers was in the 60s, when [ALGOL 60](https://en.wikipedia.org/wiki/Algol_60) had just been invented. We tediously punched lots of cards a deck of which was transferred to an operator, and a week later the result was handed back, maybe with a typo.

The same IBM mainframe also understood [Fortran](https://en.wikipedia.org/wiki/Fortran), so that was my second language. The Department of Theoretical Physics owned the second computer of the whole university, a [Zuse Z25](https://en.wikipedia.org/wiki/Z25_(computer)), which was fed with telex ticker tape. Here I learned my first and only machine language.

My next encounter with computers was in the 80s, when personal computers hit the market. I was in sales and services then. 10 years later, I led a team which developed a professional program for Windows with VC++ which was later rewritten in Delphi. At the end of the 90s, the Internet, PHP and MySQL were hot. But still there was next to no communication online except for mailing lists, at least.

I remember, on the MySQL mailing list in 1998, a guy who replied nearly every question in a very terse style. He signed as engineer at TCX AB, and I was wondering what his boss may say to his spending his time on the mailing list. His name was [Monty Widenius](https://de.wikipedia.org/wiki/Michael_Widenius).

The whole open source movement is something nobody could have ever dreamed of. Even in the new century, the invention of `git`, `github` and `stackoverflow` changed the game considerably again. 

All this is reality and somehow I am part of it. Absolutely amazing.