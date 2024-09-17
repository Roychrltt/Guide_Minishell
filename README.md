# Guide For Minishell  

This is a simple guide for minishell with only the wildcard bonus (110/100). I added my header file here, in the hope that it would make my explanations more clear.  
(I won't go into detail on certain things since this guide is supposed to only help you find the direction when you feel blocked somewhere. But I'll give hints as I could.)  
  
So there are three main parts: parsing, builtins, and execution (fork and redirection things blahblahblah).  
### 1. Parsing  
   - expand environment variables for the first time  
   - cut input string into tokens  
   - get token type  
   - deal quotes and expand environment variables for the second time  
### 2. Builtins  
   - echo  
   - cd  
   - pwd  
   - env  
   - export  
   - unset  
   - exit  
### 3. Execution  
   - pipes and forks  
   - redirection  
   - execution  
 ### 4. Others  
   - signals  
   - wildcards  

## Preparation at beginning  
  Before the parsing, I did several things to initialize my main struct `t_mem`. Following are some that might worth a brief mentioning:  
  - Use dup() to save the stdin and stdout so that you can still use the standard input or output during execution (for example, to get input for heredoc) even if you have used dup2() to redirect them, and by the end of the loop they can be redirected back.  
  - Create a linked list for environment varibales. Personnally I made 2 since the output of export is sorted. Later when we do expansions we should use this list as reference instead of `char **envp` because we might modify the environment of our minishell but the `char **envp` will apparently not change accordingly. The struct `t_env` that I used have a variable `is_unset` which indicates if this varaible still exists.  
  - Use getenv() and ft_split() to get the paths for the search of command later. (Make sure that your minishell will not segfault if we do `env -i ./minishell`)  

 
## I. Parsing  
**1.0 Check if input is valid (Are all quotes well closed? Is there a meta char by the end?)**  
  
**1.1 Expand environment variables for the first time**  
  Yes, you should try to expand environment variables in the input **before tokenizing it**.
  Example:  
  ```
  minishell$> export HELLO="ho hello"  
  minishell$> ec$HELLO  
  minishell$> hello  
  minishell$>    
  ```
  The output should be `hello` instead of `ec$HELLO command not found`  
    
**1.2 Cut input string into tokens**  
  This step could be quite annoying, expecially when you realise that the "tokens" in the command line are not necessarily separated by white spaces (or anything).  
  For example, `echo -n message>file|cat file` will work just as fine as `echo -n message > file | cat file`.
  What's more, white spaces in quotes should be kept. So we can't simply split the input using ft_split().  
  A hint for the splitting is that you can focus on **meta chars** like `>` `<` `>>` `<<` `|`. Outside of quotes they should be reliable marks of splitting.  
  
**1.3 Get token type**  
  After splitting the `char *input`, I created a linked list of struct `t_token` to store all tokens in order, while determining the type for each node at the same time. We still rely on **meta chars** in this step. For example, the token after `<` is for sure the name of the infile, and token after `<<` the LIMITER marking the end of heredoc, etc. The first string after redirections should be the command.  
  
**1.4 Deal quotes and expand environment variables for the second time**  
  Simply put, we remove the quotation marks and $VARIABLES in the arguments. Here are some examples that you should deal with:  
  ```
  minishell$> echo "'this is a message from $HOME'"  
  minishell$> 'this is a message from /Users/roychrltt'  
  minishell$> "echo" '"single quotes prevent us from interpreting things     $USER"'  
  minishell$> "single quotes prevent us from interpreting things     $USER"  
  minishell$> echo ""''""$HOME"""hi$abcauegyfe"  
  minishell$> /Users/roychrltthi
  ```  
  ft_substr() and ft_strchr() could help a lot. Be careful with uninitialised value and segfault in this step.  

**By far, all arguments should be ready to be used.**  
  
## II. Builtins  
**I would describe briefly how these builtins should behave, but I highly recommend you to do you own tests to truly learn about the behavior of these commands in `bash --posix`**  

**2.1 echo**  
  Print arguments one by one, each separated by a space. If the first argument is not `-n`, add a `\n`at the end. Otherwise, no need of `\n`.  

**2.2 cd**  
  - cd without argument would bring you back to $HOME  
  - cd with only one argument will do chdir() (this function returns -1 on failure)  
  - cd with 2 or more arguments will print an error message `cd: too many arguments` without doing chdir() for the first argument.  
  Don't forget to update $PWD and $OLDPWD in your environment variables.  

**2.3 pwd**  
  Print working directory. I used `getcdw(NULL, 0)` to retrieve the pwd.  

**2.4 env**  
  Print all environment variables. (All variables whose `is_unset` value is 0)

**2.5 export**  
  - export without argument would print all environment variables sorted by their name in ascii order. (But this is actually an undefined behavior, you can choose not to manage this at all)  
  - export with arguments would try to add each into the environment. If a variable's name is not valid (an environment variable's name can only have letters, numbers, and _, and it cannot start by a number), bash would print an error message and carrying on with the following.  
  - If one variable with the same name already exists, export this variable will rewrite its value.
  Example:  
```
  minishell$> export aaa=000 bbb 3cc ddd===  
  minishell$> export  
  bash: export: `3cc': not a valid identifier  
  minishell$> export  
  ...
  export aaa="000"  
  export bbb  
  export ddd="=="  
  minishell$>  
```

**2.6 unset**  
  - unset without argument or a variable that does not exist will not do anything  
  - unset a variable whose name is invalid will print an error message (the same as export)  
  - unset an existing variable will delete it  

**2.7 exit**  
  - exit without argument will exit minishell with exit status 0 (EXIT_SUCCESS)  
  - exit with numeric argument will exit minishell with exit status of this number (if negative, we add 256 to it until it's positive, if larger than 256, we subtract 256 from it until it's smaller than 256)  
  - if the argument is larger than LONG LONG INT MAX or contains characters other than numbers, bash will still exit but does not modify the exit status.
  - In the case of more than 1 argument, bash will print an error message without exiting
  Example:
```
  minishell$> exit 255  
  exit  
  > echo $?  
  255
```

## III. Execution  
**I assume you did pipex and understand how the pipe and fork work**  
