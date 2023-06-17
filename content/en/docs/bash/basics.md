---
title: "Basics"
linkTitle: "Basics"
weight: 30
description: >
  Overview of key terms, control structures, etc.
---


## Running a script

Before you can run a script, you have to set the execution bit on it with `chmod`:

```bash
$ chmod +x script_name
$ ./script_name
```


## Functions

There are a number of ways to define functions in bash. You should use the function keyword and no parentheses to emphasize that Bash functions do not accept paramters. Additionally, you can put the curly braces on separate lines. Make sure that you define a function before you can use it:

```bash
# signature options:
function Helper() {}
function Helper {}
Helper () {}

# preferred
function func_name {
    # commands
}
```
## Control flow

The following sections describe basic control flow syntax.

### if...then...

End the `if` block with `fi` (_if_ spelled backwards):

```bash

# if...then

if [ test_command ]
then
    # command
fi

# if...then...else

if [ test_command ]
then
    # command
else
    # command
fi

# if...then...elif...(else)

if [ test_command ]
then
    # command
elif
    # command
else
    # command
fi
```

### Loops

Indicate that a loop is complete with the `done` keyword:

```bash

# for...in
for loop_var in arg_list
do
    # commands
done

# while
while test_condition_is_true
do
    # commands
done

# until
until test_condition_is_true
do
    # commands
done
```

### Case statement

The `case` statement is the Bash equivalent to the `switch` statement in other languages. Indicate the end of command execution for each case with double semi-colons (`;;`). The default ("fall-through") case is indicated with the wildcard (`*`) character:

```bash
case $var in

match_1)
    # commands for match_1
    ;;
match_2)
    # commands for match_2
    ;;
match_3)
    # commands for match_3
    ;;
*)
    # commands for default (no match)
    ;;
```

### Control keywords

You can use the following keywords when you want to control loop execution apart from the conditions:

break
: Terminates the execution of a loop.

continue
: Passes control of a loop to the next iteration.

exit
: Exits the entire script.

return
: Sends back data, result, or return code to the calling script.

## Here documents

A _here document_ provides values to a script without user action. It is a multiline string or file literal that sends input streams to other commands and programs.

The here document must contain data in the exact format that the script expects, or it fails. For example, if there are extra spaces, the script fails.

I need to revisit this. For much better examples, see the [Advanced Bash scripting guide](https://tldp.org/LDP/abs/html/here-docs.html).