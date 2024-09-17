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
  - Create a linked list for environment varibales. Personnally I made 2 since the output of export is sorted. Later when we do expansions we should use this list as reference instead of `char **envp` because we might modify the environment of our minishell but the `char **envp` will apparently not change accordingly.  
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
## II. Builtins  
**I would describe briefly how these builtins should behave, but I highly recommend you to do you own tests to truly learn about the behavior of these commands in `bash --posix`**  
**2.1 echo**  
