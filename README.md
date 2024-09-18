# Guide For Minishell  

This is a simple guide for minishell with only the wildcard bonus (110/100). I added my header file here, in the hope that it would make my explanations more clear.  
(I won't go into detail on certain things since this guide is supposed to only help you find the direction when you feel blocked somewhere. But I'll give hints as I could.)  
  
So there are three main parts: parsing, builtins, and execution (fork and redirection things blahblahblah).  


## Preparation at beginning  
  Before the parsing, I did several things to initialize my main struct `t_mem`. Following are some that might worth a brief mentioning:  
  - Use dup() to save the stdin and stdout so that you can still use the standard input or output during execution (for example, to get input for heredoc) even if you have used dup2() to redirect them, and by the end of the loop they can be redirected back.  
  - Create a linked list for environment varibales. Personnally I made 2 since the output of export is sorted. Later when we do expansions we should use this list as reference instead of `char **envp` because we might modify the environment of our minishell but the `char **envp` will apparently not change accordingly. The struct `t_env` that I used have a variable `is_unset` which indicates if this varaible still exists.  
  - Use getenv() and ft_split() to get the paths for the search of command later. (Make sure that your minishell will not segfault if we do `env -i ./minishell`)  

 
## I. Parsing  
### **1.0 Check if input is valid (Are all quotes well closed? Is there a meta char by the end?)**  
  
### **1.1 Expand environment variables for the first time**  
  Yes, you should try to expand environment variables in the input **before tokenizing it**.
  Example:  
  ```
  minishell$> export HELLO="ho hello"  
  minishell$> ec$HELLO  
  minishell$> hello  
  minishell$>    
  ```
  The output should be `hello` instead of `ec$HELLO command not found`  
    
### **1.2 Cut input string into tokens**  
  This step could be quite annoying, expecially when you realise that the "tokens" in the command line are not necessarily separated by white spaces (or anything).  
  For example, `echo -n message>file|cat file` will work just as fine as `echo -n message > file | cat file`.
  What's more, white spaces in quotes should be kept. So we can't simply split the input using ft_split().  
  A hint for the splitting is that you can focus on **meta chars** like `>` `<` `>>` `<<` `|`. Outside of quotes they should be reliable marks of splitting.  
  
### **1.3 Get token type**  
  After splitting the `char *input`, I created a linked list of struct `t_token` to store all tokens in order, while determining the type for each node at the same time. We still rely on **meta chars** in this step. For example, the token after `<` is for sure the name of the infile, and token after `<<` the LIMITER marking the end of heredoc, etc. The first string after redirections should be the command.  
  
### **1.4 Deal quotes and expand environment variables for the second time**  
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

### **2.1 echo**  
  Print arguments one by one, each separated by a space. If the first argument is not `-n`, add a `\n`at the end. Otherwise, no need of `\n`.  

### **2.2 cd**  
  - cd without argument would bring you back to $HOME  
  - cd with only one argument will do chdir() (this function returns -1 on failure)  
  - cd with 2 or more arguments will print an error message `cd: too many arguments` without doing chdir() for the first argument.  
  Don't forget to update $PWD and $OLDPWD in your environment variables.  

### **2.3 pwd**  
  Print working directory. I used `getcdw(NULL, 0)` to retrieve the pwd.  

### **2.4 env**  
  Print all environment variables. (All variables whose `is_unset` value is 0)

### **2.5 export**  
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

### **2.6 unset**  
  - unset without argument or a variable that does not exist will not do anything  
  - unset a variable whose name is invalid will print an error message (the same as export)  
  - unset an existing variable will delete it  

### **2.7 exit**  
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
**I assume you did pipex and understand how pipe(), dup2(), and fork() work**  

### **3.1 initialize the struct for commands**  
  In this part, I have a struct `t_cmd`, in which the name of each variable announces clearly what it is for. There's also an array of int `pid_t *pids` in `t_mem` to store the pid of each child process so that I can `waitpid()` by the end of execution.  
  So, We'll first create a pipe for each command except the last using the `int fd[2]`.  
  We'll then try to get the absolute path `char *command` for the command (if it's not a builtin) and put all arguments in an array of strings `char **args`.  
  Lastly, we do the prep for redirection and store the file descriptor of input file and output file in `int infile` and `int outfile`.

### **3.2 redirection**   
  There might be more than 1 input file and output file. We should open all of them accordingly while each time closing the previous one before opening a new one. For heredoc, I would create a temporary file `.here_doc.tmp` and save the input in this file using get_next_line(saved_stdin). When it's not needed anymore, I do unlink() to remove it.  
  We get fds in the parent process but do the redirection at the beginning of each child process. If there are input files, we `dup2(infile, STDIN_FILENO)`, and `dup2(outfile, STDOUT_FILENO)` for output files.  In the parent process, we close the writing end of the pipe and redirect `STDIN_FILENO` to the reading end of the pipe. Don't forget to close all fds that are not needed anymore in both processes.  
