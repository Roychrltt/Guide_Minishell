# Guide For Minishell  

This is a simple guide for minishell with only the wildcard bonus (110/100). I added my header file here, in the hope that it would make my explanations more clear. 
  
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
  For example, `echo -n message>file|cat file` works just as fine as `echo -n message > file | cat file`.
  What's more, white spaces in quotes should be kept. So we can't simply split the input using ft_split().
