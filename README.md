# Шпаргалка по Expect  

## Полезные ссылки  

-  Книга Exploring Expect by Don Libes (рутрекер)
-  https://tcl.tk/man/tcl8.5/TclCmd/contents.htm 
-  [Grouping with double quotes.](https://www.tcl.tk/man/tcl8.5/tutorial/Tcl3.html)
-  [Tcl regex syntax references.](http://www.tcl.tk/man/tcl8.5/TclCmd/re_syntax.htm#M66)
-  [Expect glob and other patterns](https://www.oreilly.com/library/view/exploring-expect/9781565920903/ch04.html)
-  [Regular Expressions](https://www.oreilly.com/library/view/exploring-expect/9781565920903/ch05.html)
-  [Уроки Tcl](http://tclstudy.narod.ru/tcl/index.html)

## Пример интеграции в bash
```bash

#!/bin/bash
export pass="xxx"

expect <<'END_EXPECT'
    spawn ssh -o StrictHostKeyChecking=no shyamkarthikv@192.168.57.33
    expect { 
        "*?assword:" {
            send "$env(pass)\r"
            exp_continue
        }
        somePatternThatMatchesYourPrompt
    }
    set timeout -1  ;# in case it takes a long time to complete
    send "sudo installer -pkg /tmp/FS-Agent/FS-Agent.pkg -target / \r"
    expect { 
        "*?assword:" {
            send "$env(pass)\r"
            exp_continue
        }
        somePatternThatMatchesYourPrompt
    }
    set timeout 2
    send "exit\r"
    expect eof
END_EXPECT
```

## ssh login
```tcl
#!/usr/local/bin/expect -f

# debug
# exp_internal [-info] [-f file] value Управление выводом диагностической информации о полученных данных и сопоставлении с образцом. Отображение на стандартный вывод включено, если числовое значение параметра не равно 0, и отключено, если оно равно 0.
# Вывод можно отправить в файл с помощью параметра -f и аргумента имени файла.
# Опция -info вызывает отображение текущего состояния диагностического вывода.
# log_user -info|0|1
# Управление записью диалога send/expect в стандартный вывод. Аргумент 1 включает ведение журнала, а 0 отключает его. Без аргументов или опции -info отображает текущую настройку.
exp_internal            0
log_user                0
# Счётчики для повторных попыток (подключение ssh, пароль, ожидание подключения клиента л2тп на микротик)
set connect_counter     3
set passw_retry_count   3
# Логи
# set directory           "/usr/local/www/cg/bash_scripts"
# log_file -a             $directory/expect_$IPaddress.log
# prompt консоли. Варианты "#", "{[#>$] }" 
set prompt              "(] > )$"
# устранение проблем для экспекта с специмволами микротика https://help.mikrotik.com/docs/display/ROS/Command+Line+Interface
set login               "admin+cte170w300h"
set IPaddress           [lindex $argv 0]

# lindex список индекс
# 	Возвращяет элемент списка на позиции индекс. Нумерация начинается с 0,последний элемент можно указывать индексом end.
# llength список
#  	Возвращает количество элементов в списке.

# Argc - содержит счетчик количества аргументов arg (если ни одного, то 0), исключая имя файла со скриптом.
# Argv - содержит список Tcl, элементами которого являются аргументы arg, в порядке их следования, или нулевую строку, если нет ни одного аргумента.
# argv0 - содержит fileName, если он был задан. В обратном случае содержит имя, при помощи которого был вызван tclsh.

if { [llength $argv] != 1 } {    
    send_user "Usage:   $argv0 <mikrotik ip-address> \n"
    exit 1
}

set passfile "/usr/home/sergey/cg_passwords/switch.txt"

if { [file readable $passfile] } {
    gets [open "$passfile" r] password
} else {
    send_user "ERROR: The file $passfile does not exist or is not readable\n"; exit 1
}

# proc connectssh {user host} {
#     global spawn_id
#     spawn ssh -q -o "ConnectTimeout=5" -o "StrictHostKeyChecking=no" -o "UserKnownHostsFile=/dev/null" $user@$host
# }

set timeout 10

# -q может создавать проблемы для expect, выдавая лишнюю информацию, но и глушить ошибки - неправильный пароль, неверный ключ ssh и т.д.
# spawn ssh -q -o "ConnectTimeout=5" -o "StrictHostKeyChecking=no" -o "UserKnownHostsFile=/dev/null" $login@$IPaddress
spawn ssh -o "ConnectTimeout=6" -o "StrictHostKeyChecking=no" -o "UserKnownHostsFile=/dev/null" $login@$IPaddress

expect {
    timeout { 
        send_user "ERROR: Failed to get password prompt $IPaddress\n"
        exit 1
    }
    eof {
        # В некоторых сценариях catch wait result может вешать скрипт полностью. Полезность статуса сомнительна
        catch wait result
        send_user "ERROR: connection to $IPaddress failed EOF: exit code = [lindex $result 3]\n"
        exit [lindex $result 3]
    }
    # ssh 
    -nocase "yes/no" {
        send "yes\r"
        exp_continue
    }
    -nocase "Host key verification failed" {
        # exec ssh-keygen -R $IPaddress
        # connectssh $login $IPaddress
        # exp_continue
        send_user "ERROR: SSH connection to $IPaddress failed.\nHost key verification failed.\nUse ssh-keygen -R $IPaddress to create SSH key pairs\n"
        exit 1
    }
    # Это выше логина и пароля, иначе уйдет в цикл
    -nocase -re "fail|incorrect|denied|refuse" {
        send_user "ERROR: Login to $IPaddress failed. The username or password is incorrect\n"
        exit 1
    }
    # Если требуется ввод логина 
    -nocase "username:" {
        send -- "$login\r"
        exp_continue
    }
    -nocase "password:" {
        # Защита от ухода в цикл, если используется опция ssh -q или fail|incorrect|denied|refuse не сработал
        if { [incr passw_retry_count -1] == 0 } {
            send_user "ERROR: Login to $IPaddress failed. The username or password is incorrect\n"
            exit 1
        } else {
            send -- "$password\n"
            exp_continue
        }
    }
    -re $prompt {
    }
}

set timeout 5
puts "Connected"

send -- "quit\r"

expect eof
exit 0
```

## exp_continue
[exp_continue wiki](https://wiki.tcl-lang.org/page/exp_continue)
By default, exp_continue resets the timeout timer. The timer is not restarted, if exp_continue is called with the -continue_timer flag.

## Получение статуса и выход с этим статусом. 
```tcl
spawn true
expect eof
catch wait result
exit [lindex $result 3]
```
Exits with 0.

```tcl
spawn false
expect eof
catch wait result
exit [lindex $result 3]
```

Exits with 1. 

## Вывод полного буфера 
```tcl
#!/usr/bin/expect
match_max 10000
spawn dbscontrol
expect "QUIT:"
send "display\r"
send "quit\r"
set log [open "dbscontrol.general.dump" "w"]
set NewLineChar "\r"
expect {
    $NewLineChar { append dbscntl $expect_out(buffer); exp_continue}
    eof { append dbscntl $expect_out(buffer) }
}
puts $log $dbscntl
```

### Intro

TCL-Expect scripts are an amazingly easy way to script out laborious tasks in the shell when you need to be interactive with the console. Think of them as a "macro" or way to programmaticly step through a process you would run by hand. They are similar to shell scripts but utilize the `.tcl` extension and a different `#!` call.

### Setup Your Script

The first step, similar to writing a bash script, is to tell the script what it's executing under. For `expect` we use the following:

```
#!/usr/bin/expect
```

You also must place `interact` at the end, so your script looks like this:

```
#!/usr/bin/expect



... All your codez...



interact
```


### Puts & Output

Instead of the `echo` command, expect uses puts, which is pretty 1:1...

```
puts "I am performing a command..."
```

Would do exactly what you think it would, just echo out the text.

Now, the script will also show you the commands being performed. Good for testing, but maybe not required in "production" or daily-use. In that case you can just show the `puts` text via:

```
log_user 0
```

Which suppresses the commands and responses, showing only what you output via `puts`. You can always turn it back on later via:

```
log_user 1
```

If for example you wanted to show exactly what is coming back to the console.


### Variables

Variables are very simple, just use `set {name} {value}`, for example:

```
set a "apple"
set b "banana"
set c "cantalope"
```

Referencing variables is done by simply appending `$` to the name - so `$a` would contain `apple`.


### Arguments

If you want to set a variable by passing in an argument simply use the following:

```
set user [lindex $argv 0]
set password [lindex $argv 1]
```

Then, when running the script I could supply the varaible content via:

```
./myscript.tcl myuser mypassword
```

### Spawn & Send

You can start a process with the `spawn` command. For example, `ssh` or `scp` are great examples:

```
set user "myuser"
set server "myserver.com"

spawn ssh "$user\@server"
```

The above would spawn the ssh process and submit the user and server, so essentially like entering `ssh myuser@myserver.com` in the console.

The `send` command allows you to send something to the console. For example, a password:

```
set password [lindex $argv 0]
set app [lindex $argv 1]

spawn sudo "apt-get install $app"

expect "assword"

send "$password\r"
```

The above would allow you to pass in your sudo password and the name of an application to install from apt, then wait for the password prompt and send it. No `assword` is not a mispelling, let's look at that `expect` statement...

### Expect

The `expect` statement is where the magic happens. Let's start with the basic, say you run a command and just want to wait for the console to return to prompt before it moves on...

```
spawn apt-get "update"

expect "$"

# Move on to the next thing...
```

Very simple stuff, you spawn `apt-get` then just wait for the `$` (or prompt).

The expect statement looks for a "close match" so in the above example your prompt (depending on your shell) could be something like `root@server $~`. The expect would find that `$` and know that it's good to move forward.

So, how about more difficult cases, where you may not have a finite expect. Let's take a look at ssh again:

```
spawn ssh "$user/@server"

expect {
    "assword" {
        send "$password\r"
    }
    "yes/no" {
        send "yes\r"
    }
}

# NOW we can move on...
```

In the above example we could encounter two results. The first is that I have connected to the server previously and am prompted with `Password:` or `Enter your password for server:` (or something similar). The looseness of the `assword` define covers both cases and submits the `$password` variable. The `"yes/no"` would be if you're prompted to accept the remote host's key - it will submit `yes` for you, then wait for the password-prompt and handle that for you as well.

This can be adapted to different prompts as well - you could do something like the following to account for different styles of prompts:

```
expect {
    "> " { }
    "$ " { }
}
```

This can be built-upon more to really close the error-gap:

```
expect {
    "> " { }
    "$ " { }
    "assword: " { 
        send "$password\n" 
        expect {
            "> " { }
            "$ " { }
        }
    }
    "(yes/no)? " { 
        send "yes\n"
        expect {
          "> " { }
          "$ " { }
        }
    }
    default {
        send_user "Login failed\n"
        exit
    }
}
```

### Conditionals

If-Statments are extremely simple, and an example should be clear enough to make the point:

```
if { $a == "soup" } {
    puts "You have soup!"
else {
    puts "No soup for you!"
}
```

### Looping

There are several methods for looping. I tend to take the `while` approach when possible:

```
set count 10;
while {$count > 0 } {
    puts "$count\n"
    set count [expr $count-1];
}
```

The above will simply loop backwards from `10` and echo-out the number.

### Functions & Proc

What about code re-use? Oh yeah - expect has that:

```
proc do_something { a b c } {
    puts "You should buy some $a, $b, and $c!\n"
}
```

Can then be called with:

```
set running [do_something "apples" "bananas" "cantalopes"]
```

Which would output:

```
You should buy some apples, bananas, and cantalopes!"
```

### Conclusion

The above describes the basics of expect scripting. It's really simple when you think about it in terms of taking actions you would perform by hand and converting them into a scripted set of actions.

I decided not to give some big example but rather cover the core concepts. I did this because when I got started the examples I saw just threw me off and once I spent some time digging to understand the fundamentals I quickly found myself writing the scripts with ease. Hope this helps you do the same!
