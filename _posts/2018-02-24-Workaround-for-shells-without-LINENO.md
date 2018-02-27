---
layout: post
title: Workaround for shells without LINENO
published: true
---

## Table of Contents

- [Workaround for shells without LINENO](#workaround-for-shells-without-lineno)
- [Debugging](#debugging)
- [More complex cases](#more-complex-cases)
- [How to use](#how-to-use)
- [Caveats](#caveats)
- [Logging](#logging)
- [Example](#example)
- [Why roll your own?](#why-roll-your-own)
- [Error handling](#error-handling)
> - [Repair or copy (digression)](#repair-or-copy-digression)
> - [Table partitioning (digression)](#table-partitioning-digression)
> - [Partition by md5 (digression)](#partition-by-md5-digression)
> - [PRIMARY KEY clause (digression)](#primary-key-clause-digression)
> - [Unusable partition distribution (digression)](#unusable-partition-distribution-digression)
> - [Experimenting with CONV (digression)](#experimenting-with-conv-digression)
> - [Max value of bigint datatype (digression)](#max-value-of-bigint-datatype-digression)
> - [Table type: MyISAM vs. InnoDB (digression)](#table-type-myisam-vs-innodb-digression)
- [Regular health checking](#regular-health-checking)
- [Automatic failover](#automatic-failover)
- [Adding a stopwatch](#adding-a-stopwatch)
> - [Docker and mysqlbinlog (digression)](#docker-and-mysqlbinlog-digression)
- [Why roll your own, revisited](#why-roll-your-own-revisited)
- [Have fun](#have-fun)

Workaround for shells without LINENO
----------

Define a function

    echo_line_no () {
        grep -n "$1" $0 |  sed "s/echo_line_no//g" 
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

Debugging
----------

Lucky are those who have bash, but ash does not have LINENO, alas. Which is bad if you need to do a lot of testing on `Busybox` or the like.

That's why I finally developed a simple and powerful workaround with interesting side effects for debugging and logging purposes. 

Instead of using 

    echo $LINENO this is a simple comment with a line number

which does not work with e.g. `Busybox` or `Boot2Docker` or `Tiny Linux`, introduce a small and simple function `echo_line_no` (or whatever function name you like better) which prints the line number pretty much, but not exactly like LINENO. 

Line numbers are not just for fun, they are badly needed for debugging, and that's where things get more complicated. 

I could just leave all of that as an exercise to you, but this is not the intention of Stack Overflow as I understand it. Hence I elaborate on the subject to give you as complete an insight as is possible for me. 

Sorry, it gets a bit lengthy for that reason. While I was developing my ideas, I digressed a bit into database specifics new to me. If you are not interested, please simply skip those sections.

In case you fear I get talkative or be prating, just stop reading. 

In particular I don't want to insult all you experts who are better than I, have far more experience and can do all of this on their own. 

It's for people like me I take the pain to write this all up, those who are new to the subject and have to fight their way more or less alone. Kind of paying back my debt according to the old mailing list ethics. In addition it turned out that this study revealed a lot of insight for myself as well.

Always bear in mind that I can only talk from my own experience, which is limited. So take this text with a grain of salt.

More complex cases
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
        grep -n "$input" $0 |  sed "s/echo_line_no//g" 
    } # echo_line_no

The result for the enhanced version using the test script `test_echo_line_no.sh` (see below) is:

    $ /path_to_your_script/test_echo_line_no.sh
    16   "this is a simple comment with a line number"
    18   "ok for me"
    20   "ok for you"
    -- without quotes (grep "$1") will filter for "ok" only, giving too many (multiple) results
    24   "this is a multiline comment, will be cut off at new line
    29   "this is another simple comment with line number and variable FOO enclosed :$FOO:"
    27  FOO=bar
    -- you can always show the value of a variable and the line it is defined
    14  msg='a simple message'
    37  msg='another simple message'
    -- simple call inside function will try to filter for the argument
    ==:hey:====== argument given to function whatsup ========
    43  whatsup "hey"
    11       "this was from inside function whatsup, argument :$1:, line number is call line"
    -- without VARTOKEN results in not showing anything
    ==:howdy joe:====== argument given to function whatsup ========
    11       "this was from inside function whatsup, argument :$1:, line number is call line"
    45  buddy=joe
    ==:hi my dear buddy :joe::====== argument given to function whatsup ========
    53  whatsup "hi my dear buddy :$buddy:"
    11       "this was from inside function whatsup, argument :$1:, line number is call line"

How to use
----------

Create a script defining the function only. This script is to be included in the real script to be debugged:

    #!/bin/ash
    #/path_to_your_script/echo_line_no.sh  
    
    echo_line_no () {
    #echo "--echo_line_no input :$1:--"
    # to see what is coming in
        input=$1
        NL='
    '
        input=${input%%"$NL"*}
        VARTOKEN=:
        input=${input%%$VARTOKEN*}
        grep -n "$input" $0 |  sed "s/echo_line_no//g" 
    } # echo_line_no

Include this script into your working script (which you want to debug) via `source` call 

    source /path_to_your_script/echo_line_no.sh 

In consequence, you can use the function `echo_line_no` in your testing script anywhere you like, in particularly inside functions.

Here is the script used for testing the functionality of `echo_line_no` whose output was shown above:

    #!/bin/ash
    #/path_to_your_script/test_echo_line_no.sh  
    
    source /path_to_your_script/echo_line_no.sh
    
    function whatsup {
    echo "    ==:$1:====== argument given to function whatsup ========"
        echo_line_no "$1"
        # quotes are crucial -- otherwise 'hi my ...' 
        # would give all lines with "hi" like "this is...."  
        echo_line_no "this was from inside function whatsup, argument :$1:, line number is call line"
    } # whatsup 
    
    msg='a simple message'
    
    echo_line_no "this is a simple comment with a line number"
    
    echo_line_no "ok for me"
    
    echo_line_no "ok for you"
    
    echo '    -- without quotes (grep "$1") will filter for "ok" only, giving too many (multiple) results'
    
    echo_line_no "this is a multiline comment, will be cut off at new line
    second line of comment with a line number"
    
    FOO=bar
    
    echo_line_no "this is another simple comment with line number and variable FOO enclosed :$FOO:"
    
    echo_line_no "$FOO"
    
    # you might omit the quotes here because the variable contains no blanks

    echo '    -- you can always show the value of a variable and the line it is defined'
    
    echo_line_no "$msg"
    
    # mind the quotes -- the variable contains blanks
    
    msg='another simple message'
    
    # mind the quotes -- the variable contains blanks
    
    echo "    -- simple call inside function will try to filter for the argument"
    
    whatsup "hey"
    
    buddy=joe
    
    echo "    -- without VARTOKEN results in not showing at all"
    
    whatsup "howdy $buddy"
    
    echo_line_no "$buddy"
    
    whatsup "hi my dear buddy :$buddy:"

Caveats
----------

1. Remember, all the magic stems from `grep`. Make sure each input string is significantly different to any other line in the script, as this is the token for `grep` -- otherwise you get more than that single line you want to see.

2. Examples 2 and 3 show how important quotes are for the argument to the function -- try it without quotes to see the effect (`echo_line_no "$1"`). Without quotes, only the first word is the trigger which will find 2 lines on each call here, so you get 4 results instead of 2, which will most likely be confusing.

3. For multi-line strings, this constraint of uniqueness applies to the first line only as `grep` is line oriented -- the argument however has more than just one line, so grep will fail and you see nothing unless we cut off everything after the first line. Consequently you will not see the other lines in the output, but that may not be really bad unless you need the information therein; if this is a problem, consider putting the information you need into the first line.

4. For the use of variables, this constraint of uniqueness applies to the part up to the token character `VARTOKEN` (here `:`) used to enclose these (which is a good idea anyway to see if a variable is empty, see example) -- reason: `grep` looks for the original line and will not recognize the substitution (which we do not know), so the code has to stop here as well. This is no real problem as you can always use `echo_line_no` on the variable itself to get the line number where it is called (see next example). 

5. For the next 3 examples, you will not get the line number of the comment, but the line number of the definition of the variable instead -- which may be just what you want as this information is hard to find otherwise. The first one shows said variable missing from the function call (example 4), the next 2 assignments show the lines of definition of the same variable defined at different places with different values.

6. You can use `echo_line_no` from inside a function, but in contrast to bash there is no way to get the function name via system variable due to the same restrictions. No problem, you can always hardcode, as demonstrated here; you have to hardcode the call to `echo_line_no` there anyway. But there is more to the use within functions:

   a. If the function argument does not contain a variable, the function shows the line number of the function call, like with variables. That's kind of a trace function, you see which line has called your function -- again something most valuable and hard to get otherwise. The line of the call within the function would be useless anyway.

   b. If it does contain a variable and this variable is not masked by `VARTOKEN`, the whole line is not shown at all as as seen in the 2nd function example, as `grep` must fail due to variable substitution; in order to compensate this, call `echo_line_no` also for the variable itself to get the line number, as demonstrated.

   c. If you use `VARTOKEN`, the line number is shown as in the 3rd function example; this example also shows that inside the function quotes are crucial as well due to the same reason, but in special cases you may even be interested in other places your first word appears (try it without quotes to see the result). Also, if a comment repeats the trigger, it will be shown, too, that's why the comment inside this function has been crafted carefully to not fall into this trap. You will notice anyway and know what to do, if it happens by chance.

Logging
----------

You may even write a protocol for later inspection or tracking the performance of the working script. To this end simply add the appropriate instruction to your function, e.g. 

    | tee -a /tmp/echo_line_no.log

like so:

    grep -n "$input" $0 |  sed "s/echo_line_no//g" | tee -a /tmp/echo_line_no.log

The log file in this case -- no surprise -- looks like

    $ tail -f -n 60 /tmp/echo_line_no.log
    16   "this is a simple comment with a line number"
    18   "ok for me"
    20   "ok for you"
    24   "this is a multiline comment, will be cut off at new line
    29   "this is another simple comment with line number and variable FOO enclosed :$FOO:"
    27  FOO=bar
    14  msg='a simple message'
    37  msg='another simple message'
    43  whatsup "hey"
    11       "this was from inside function whatsup, argument :$1:, line number is call line"
    11       "this was from inside function whatsup, argument :$1:, line number is call line"
    45  buddy=joe
    53  whatsup "hi my dear buddy :$buddy:"
    11       "this was from inside function whatsup, argument :$1:, line number is call line"

This approach is quick and easy, but not good enough. I need to log to different log files, so I'd like to add a switch to define the log file name -- if given -- instead of a fixed log file for all use cases no matter what. 

To implement this feature, first change the function script as follows:

    if [ -n $log_echo_line_no ]
    then
        grep -n "$input" $0 |  sed "s/echo_line_no//g"  | tee -a $log_echo_line_no 
    else
        grep -n "$input" $0 |  sed "s/echo_line_no//g" 
    fi
    # log_echo_line_no may be defined in calling script 

Next in your file to be debugged (here `test_echo_line_no.sh`), add the instruction

    log_echo_line_no=/tmp/test_echo_line_no.log 

before calling `source /path_to_your_script/echo_line_no.sh`. That's all.

If you also want to clear the log file when starting the test script, simply add 

    echo > $log_echo_line_no

before the first call to `echo_line_no`.

Example
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

    240 /path_to_your_script/mysql_repl_monitor.sh ---------------------------------------------- 2018-02-27_12:23:00 
    175 =====> s1 2018-02-27_12:23:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82352145
    175 =====> s2 2018-02-27_12:23:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82352145
    
    240 ----------------------------------------------------------------------- 2018-02-27_12:24:00 ---
    175 =====> s1 2018-02-27_12:24:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82424522
    175 =====> s2 2018-02-27_12:24:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82424522
    
    240 ----------------------------------------------------------------------- 2018-02-27_12:25:00 ---
    175 =====> s1 2018-02-27_12:25:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82496899
    175 =====> s2 2018-02-27_12:25:00 OK Master_Log_File mysql-bin.000006 Read_Master_Log_Pos 82496899

This is how it should be, regular entries at the given interval, nice and boring. 

If not, you see at a glance that something is wrong. Plus you see on which line of the script the output is generated and hopefully with the full error code from the replication instance reporting the error so you immediately know exactly what has happened where to which replication engine so you might take the right action based on this information in order to get replication to work flawlessly again. 

Line numbers aren't important if things are going smoothly, but if you are creating and debugging complex scripts like this one or -- more importantly -- if indeed an error occurs, you will be glad to know on the spot where to look.

So this should illustrate the importance of logging. Some final thoughts about the background of all this debugging business. 

Why roll your own?
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

Error handling
----------

We would need at least one more shell script to clean up the problem as soon as possible, and that is the moment an error is detected, which only depends on the interval of the monitoring service, itself being realized as a shell script. 

Giuseppe Maxia does not have a proposal of how to handle this problem. The classical way would be to inspect the nature of the problem first. Why did replication stop?

This is a good approach if you don't know what may happen. Maybe there is a logical or technical flaw in the application code causing this trouble. In this case it doesn't make sense to reset the slave and start again, because this error will undoubtedly appear again and again. You have to inspect your application code and make sure that no database error is induced by intention or carelessness.

But if you are sure that this doesn't happen and the error is not triggered and cannot be tackled by code, it may be a viable solution to apply brute force. If I understand the approach of database specialist company Percona correctly, this is what they do. They check at the level of the operation system if the corresponding tables are in sync, and if not, they just copy the original master table to the slave. 

Here I don't write about fancy scenarios. I have experienced replication errors which obviously stem from the database engines involved and which I could not explain. Google of course knows about these errors. They are discussed in the MySQL forum, but none of these cases has found a solution. So there is nothing I can conclude here. Brute force is the only remedy.

Repair or copy (digression)
----------

As these are the only errors I ever experienced apart from those induced by faulty code, I decided to develop a solution along the strategy of Percona. Thinking about it, it is an elegant solution as well. 

Consider for example the case that both master and slave tables differ because the slave table is corrupt for some reason (this may happen and usually nobody knows why -- could be hardware failure). 

There are 2 strategies here. Either you let the database engine of the slave repair the slave table (which you have to do anyway even if this error is not detected by sync checking). This still doesn't give you the guarantee that both tables are identical bytewise. Or you copy the master table to the slave right away.

This (repair or copy) may take a lot of time depending on the size of your tables. For this reason, you may contemplate to partition big tables into sizable chunks. 

Chances are, only one file of the partition tables has a problem. In this case you only copy the faulty file which is most probably the fastest operation you can get.

Table partitioning (digression)
----------

For example, I have more than a dozen MyISAM tables of the same type of data some of which with more than 1 GB datafile each. This will slow down rsync operations on those tables. 

Investigating this problem, I ended up dividing those tables into 3 groups, those small enough to not be partitioned, those to be partitioned into 100 chunks and into 1000 chunks. This way, I hoped to be able to keep table size in the range of at most 10 Mbytes. 

All those tables have an auto increment primary key id, so the partition formula is a kind of modulus operation given by

    ALTER TABLE tbl PARTITION BY hash (id * 100 + id) PARTITIONS 100;

or 

    ALTER TABLE tbl PARTITION BY hash (id * 1000 + id) PARTITIONS 1000;

respectively.

Looking at the biggest of those tables, only 605 of these 1000 partitions contain data so far, so the data obviously is not distributed evenly. The positive feature is that all data belonging to a particular id will be found within a single partition. As most queries ask for data correlating to a particular id, this means that only one partition table has to be opened. This justifies partitioning in the first place.

With a table of different type, but the same property of giving exact hits in case of partitions, all 1000 partitions were filled quite evenly, so the principle as such seems to be correct and viable.

The average size of the partition tables is about 2 MB. The biggest chunk, however, has about 416 MB, which isn't quite what I was heading for. The situation doesn't seem to be that bad, though. The chance to hit one of the bigger partitions actually is much lower than hitting the whole unpartitioned table.

You may wonder about the peculiar formulas for the partition definition. They are due to the fact that the integer column used for hash partitioning starts with 1, which would make all records belonging to the first 100 or 1000 id would populate the first partition table. This formula avoids that. All of these first id records now belong to different partitions.

Partition by md5 (digression)
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

PRIMARY KEY clause (digression)
----------

    >ALTER TABLE bak.tbl_md5 PARTITION BY hash (id_ct * 100 + id_ct) PARTITIONS 100;
    ERROR 1503 (HY000): A PRIMARY KEY must include all columns in the table's partitioning function

Oh yes, of course. 

When I encountered this error the first time it took quite a while for me to understand. Google was my first reach for help as usual, but everybody just said: the error message tells it all. This didn't help me much.

With a primary key as the only unique key, things may be easier to understand, but I happened to tackle a table with an additional unique key and had to comprehend that every unique key has to be changed accordingly. 

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

Unusable partition distribution (digression)
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

This is funny. It shouldn't be. In my understanding a hex value should just be a number. Let's start simple.

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

Experimenting with CONV (digression)
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

Max value of bigint datatype (digression)
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

Better not. The magical number 18446744073709551615 is just 2^64-1 and the maximum of an unsigned big int. That's why! I never hit that number before. 

Table type: MyISAM vs. InnoDB (digression)
----------

The data collected so far is just the beginning. Eventually all the partitions will be filled up quite evenly. Also, the overall size will grow accordingly, so we might have to introduce partitions by 10,000 - well, this is not possible at the moment, the limit is 8192, which, for a human only, makes it more difficult to compute the partition a particular id is to be found.

There are good reasons why I chose MyISAM as database engine for these tables and not InnoDB. The main reason of course is that we don't need any of the features discriminating InnoDB from MyISAM, not now in test mode and not later under heavy production conditions. 

With partitioned InnoDB tables, things are more complicated, as usual, but moving files around can be done nevertheless. See [Importing InnoDB Partitions in MySQL 5.6 and MariaDB 10.0/10.1](http://www.geoffmontee.com/importing-innodb-partitions-in-mysql-5-6-and-mariadb-10-010-1/)

In case of MyISAM you can even choose which method to apply if things go wrong, `repair` or `copy`. The `frm` file is not touched as a rule, so there is nothing to do. If the data file `MYD` is different between master and slave, you best copy. `REPAIR TABLE` must copy as well, so this doesn't cost you any more time. 

If the index file `MYI` is different, you best use `REPAIR TABLE`. Chances are, the repair is immediate, because very often it is just the number of records which is wrong being zero or any other number different from the correct number.

Of course you have to make sure that the master table is not changed during these procedures. And if the process succeeded, you restart the slave reporting that problem and check if everything is okay now. End of digression.

Regular health checking
----------

One more consideration here. The monitoring script can only detect errors which occur during operation. If at startup the table setup on the slave is different from the master for some reason or the slave database engine produces an error maybe due to some hardware failure, the slave will never find out and the monitoring script is of no use in this case. Comparing the files directly however will find out and the copy process will successfully synchronize any files that are not in sync.

To show you how this could be utilized by way of precaution in a running [Docker](https://de.wikipedia.org/wiki/Docker_(Software)) system, later on I show a little script `/path_to_your_script/mysql_rsync_lock.sh` which takes next to no time (in test mode) reading and writing to spinning disks if everything is okay (1-2 secs for 64 tables, about 5 GB disk space, 2 slaves), so it even might be run regularly. 

Well, this is not based on actual production data. In case your data moves quickly and indices are changed considerably in the process, the command `FLUSH TABLES` alone may take minutes. In production mode, we will use SSD and much more RAM, so those processes will be much faster. As far as we can see now, the whole database can be held in RAM.

You see, a solution to problems depends heavily on the nature of your setup.  If your system resembles Wikipedia, you have totally different problems from one that works like Facebook, eBay, YouTube or Google.

If you have more slaves, maybe even a large number, I would think of a kind of daisychain test. Compare the master with one of those slaves first, locking the master and this slave, then unlock both the master and the slave, give the slave time to catch up, then lock it again and the next slave to compare this one with the first slave, and so on. 

This way you will detect problems if replication traffic is not heavy, which means you don't have many time-consuming write processes which produce replication lag. I don't know if this will work. Data may be moving too fast for this kind of procedure. 

The benefit of this scenario obviously is that the master has to be locked for the comparison with one other machine only, no matter how many slaves you have. The slaves run in read mode anyway, and the penalty for locking is just slightly stale data, if any. The master being locked may not be acceptable at all -- it all depends on your setup.

You should utilize global transaction identifiers (GTID) if things get complicated. If you do your copy job, you know the GTID of the source, can make sure the GTID of the source is greater or equal than the GTID of the target, and can synchronize the target to exactly that same GTID before starting that slave. 

This way you can be sure to not leave any transaction out and to not duplicate any transaction which would produce trouble as sure as hell. Again, this is just an idea. If you really have severe problems, call the experts from Percona. Or develop your own solution and become an expert yourself.

For the repair process it would be intelligent to analyze the error message from the slave. If that shows that just one table is affected, then only this table has to be processed. Doing so will speed up the whole thing tremendously, in particular if you have lots of tables.

Automatic failover
----------

Having a solution for occasional errors on slave machines doesn't mean your done. What happens when the master has a problem and this problem is propagated to the slaves? Well, I'm afraid there is no automatic solution for that problem.

But if the master stops or a problem can be detected on the master and the slaves are not affected, one of those may be elevated to be a master for the other slaves. There are 2 questions here. Which of the slaves should be the new master? How to realize the failover?

Here you may feel that this whole scenario isn't just something for individual solutions crafted with shell scripts. It is a generic problem not depending on the operation system or the nature of the database.

The problem is as general as load-balancing. You wouldn't want to write a load balancer yourself. That's why I included [haproxy](http://www.haproxy.org/) as a docker container and let this container do the load-balancing work for the 3 database containers.

Just as well I could have used [MaxScale](https://mariadb.com/resources/blog/mariadb-maxscale-22-introducing-failover-switchover-and-automatic-rejoin) for this purpose, and indeed I have experimented with it in times when it was not yet mature. It's time to switch, I guess, so I will have a 2nd look soon, because in addition to load balancing, MaxScale has some more features, one of them being automatic failover. And this is something you definitely want to have if you can get it. And of course there are alternatives, too, like [Master High Availability Manager](https://www.percona.com/blog/2016/09/02/mha-quickstart-guide/).

Adding a stopwatch
----------

As I wanted to prove my claim with respect to the time taken, I noticed my chance to show you the power and elegance of Docker and improve the original debugging mechanism by adding a kind of stopwatch. 

Remember, `echo_line_no` takes exactly one parameter. If we add a 2nd parameter and this parameter is "DATE", then we take the time and show it.

The property of `echo_line_no` not showing anything if a variable is part of the first parameter, which looks like a flaw, now turns into a feature. We use the variable to suppress the output and only show the datetime. 

In the snippet below you see the 2 calls to rsync plus the time but no line number.

    docker@boot2docker:/tmp$ /path_to_your_script/mysql_rsync_lock.sh
        =========================================== 2018-02-27_00:12:57
    41: " -------- flush tables lock tables" DATE
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
    66: " unlock tables flush tables"
        =========================================== 2018-02-27_00:12:59
    75: " -------- done" DATE
    ---------------------------------------------- time taken 2 seconds


To implement this, add the first snippet to the top of `/path_to_your_script/echo_line_no.sh`, 

    TIMEDIFF=3600
    # to compensate UTC when called by cron
    
    function get_date {
        DATETAKEN=$(date --date="@$(($(date -u +%s) + $TIMEDIFF))" "+%Y-%m-%d_%H:%M:%S")
    } # get_date 

the 2nd snippet has to be put inside the function `echo_line_no`.

    if [[ ! -z "$2" && "$2" == "DATE" ]] 
    then
        get_date
        echo "    =========================================== $DATETAKEN "
    fi

This is the synchronizing script:

    #!/bin/sh
    # /path_to_your_script/mysql_rsync_lock.sh
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
    echo_line_no " -------- flush tables lock tables" DATE
    for db_server in m1 s1 s2 
    do
        docker exec $db_server mysql -e "use $db; flush tables; lock tables;"  
    done
    
    echo_line_no " -------- dat_master dat_slaves"
    for db_dir in $dat1 $dat2 
    do
        sudo rsync -av --delete $datm/$db/ $db_dir/$db/  
        # in case of slave error will eventually find something
        echo_line_no " ---$db_dir---- " DATE
        # take the datetime here, suppress line, show only datetime 
    done
    
    echo_line_no " -------- SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1"
    for db_slave in s1 s2 
    do
        docker exec $db_slave mysql -e "STOP SLAVE; SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; START SLAVE;" 
    done
    
    echo_line_no " unlock tables flush tables"
    for db_server in m1 s1 s2 
    do
        docker exec $db_server mysql -e "use $db; unlock tables; flush tables;"  
    done
    
    echo_line_no " -------- done" DATE
    
    END=$(date +%s)
    
    let USED=$END-$BEGIN
    
    echo "---------------------------------------------- time taken $USED seconds"

Here we use `docker` in combination with the `mysql` client like a function which is extremely elegant and very powerful. You can do quite complex database operations this way.

Obviously, the slave is in sync with the master after this operation. So skipping the offending operation on the slave will get the slave running again. 

At least I hope so. Maybe there is some more work to be done. The master has been locked and his binary log stopped at a certain position. Maybe we need to also synchronize the slave with respect to this position. Under heavy load you will certainly find out. Even in test condition, you can produce that load. Well, I should investigate this question. 

The script `/path_to_your_script/mysql_repl_monitor.sh` polls every minute, as you see from the replication log and `crontab -l`, which is as frequent as cron allows. The call is cheap on the respective database engines, so there is no performance problem to be expected.

Docker and mysqlbinlog (digression)
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

With the utility program `mysqlbinlog` we can read and export the binary log file to a file which is readable in parts; in parts only because SQL instructions which may contain sensible data are encrypted. The correct syntax for the docker instruction is for example

    docker exec s2 /bin/ash -c 'mysqlbinlog -r /tmp/s2.mysql-relay.000002.sql /var/lib/mysql/mysql-relay.000002'

In order to be able to inspect the protocol file from the host, you have to map your tmp directory accordingly. This you do in the docker-compose yml file for master and slaves in case you use docker compose, for example:

    volumes:
      - /c/tmp:/tmp
      - /d/data/master:/var/lib/mysql
      - /d/data/sphinx:/var/lib/sphinx/data
      - /etc/ssh/:/etc/ssh/

As you see here, we also use the [Sphinx database search engine](http://sphinxsearch.com/).

If you want to see what your database engine really does, you better record every data changing operation in a separate table. As we don't utilize MaxScale yet, we had to implement a master/slave-switch in our application. That's where the logging mechanism belongs:

    $this->_connection_type = 'db_master';
    $this->_sql_log_record($sql);

The SQL term is compressed to save space, so searching for or looking at specific queries requires uncompressing. You cannot see the queries in your conventional [Adminer](https://www.adminer.org/) interface.

MaxScale can do all that for you via [MaxScale Read-write splitting](https://mariadb.com/kb/en/mariadb-enterprise/mariadb-maxscale-21-readwritesplit/) and [MaxScale Query Log All Filter](https://mariadb.com/kb/en/mariadb-enterprise/mariadb-maxscale-14/maxscale-query-log-all-filter/). 

Rolling your own, however, you know exactly what you do. If you record all your data changing queries, that table may fill up very quickly, so you might implement a mechanism to regularly discard data as well.

This is another example for a good use of partitioning a table, this time by a `RANGE`. The benefit is, that dropping lots of records by range cost nothing, as it is done immediately by dropping that particular partition. 

This regular dropping of the oldest partition and creating a new partition instead can be realized via stored procedure (you will find examples via Google) or conventionally via crontab, shell script and docker. Take your pick. End of digression.

Why roll your own, revisited
----------

Some more considerations might be helpful. Let's again learn by example.

 
For the problem of slaves running out of sync, I already mentioned the ready-to-use solutions by "Experts in Database Performance Management" Percona ([pt-table-checksum](https://www.percona.com/doc/percona-toolkit/LATEST/pt-table-checksum.html), [pt-table-sync](https://www.percona.com/doc/percona-toolkit/LATEST/pt-table-sync.html)), but those make heavy use of Perl which may not be at your disposal and you may not be willing to install a big software packet just to connect to your database engine to begin with. So this is of no use for you. 

But there is another reason why to step back here. I concede that, at first glance, it's compelling to be lazy and use other people's programs, but you have to understand them, too, if you want to make use of them, and if these tools don't work out as expected, you will have to analyze their code anyway (if you can) and try to make sense of it and find that piece of code that doesn't do as it should (I am no expert in Perl and would not like to invest time and energy to become good enough to debug a program by Percona). 

A second example: The shell script of Giuseppe Maxia mentioned above didn't record the complete error message, but only the first word, hence his e-mail doesn't have any information about the nature of the error, as the first word in the error message is "Error", which doesn't help at all. 

Too bad and easily fixed by adding a 2nd function especially for use in the e-mail or the log file extracting not only a single value from the response of the server but the complete sentence to be used for those error messages. 

But he also uses arrays, which are handy in bash, albeit not implemented in ash. So for this reason alone his code is not usable out-of-the-box and has to be rewritten for platforms not having bash.

Nevertheless, it is great to build upon brilliant code of masters like Giuseppe Maxia, learn from them and grow along the way. Which is exactly the mission of Stack Overflow, if I understand correctly. 

Thanks a lot to everybody involved in this great and wonderful worldwide endeavor. Who would have been able to dream of this phenomenon, say, 30 years ago?

Summarizing, if you roll your own solution, 

 - you know what you're doing,  

 - your solution reflects exactly the nature of your special setup,  

 - and you're in complete control.

Again you will be glad to have a handy debugging tool like `echo_line_no` with complex tasks like database replication repair on platforms without LINENO. You will need it even more so as your solution cannot claim to be tested by a plethora of experts with all kinds of field experience.

Have fun
----------

Finally, if you want to repeat the above given tests, don't forget to replace `path_to_your_script` with your own path (or create a symbolic link, whichever is easier for you). Take this sample as a starting point for your own creativity. Maybe there are other exciting things you can do with this approach I couldn't come up with (yet). 

You see, I haven't tested this approach under heavy conditions because unfortunately I already had developed these database repair scripts mentioned above without making use of `echo_line_no` -- in fact the deficiencies in debugging these programs finally made me look for a solution. I reckoned with lots of people having the same problem and some who not only know what to do, but published solutions on StackOverflow, for example.

I finally found this page via Google "shell script display line numbers -diff -tail"; the term "shell script display line numbers -bash -diff -tail" which I used before obviously missed this entry. 

The contributions to this page so far are nearly 5 years old now. They have shown me that there is no solution for my problem except I create one myself, which I did.

Last but not least this text will be indexed by search engines and may be found for quite some time to come by people like me looking for a solution of their problems related to any of the search-relevant technical terms I have used. 

As people in times of Docker containers tend to use minimal Linux systems like Boot2Docker, CoreOS or Alpine Linux using ash instead of bash, most probably there will be more need for a substitute for LINENO. That's the path I have taken. Those 2 MySQL (or rather MariaDB) replication engines run in Docker containers as does the master. The OS is Boot2Docker which is based on Tiny Linux which in turn is based on Busybox.

Hopefully, this text will give somebody else some insight in the future as well. Furthermore, I hope that you, having read so far, did enjoy the article and don't regret having spent your time.

Last but not least I hope I didn't make a fool out of myself.