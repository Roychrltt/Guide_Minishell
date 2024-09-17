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


## I. Parsing  
1.1 Expand environment variables for the first time  
  Yes, you should try to expand environment variables in the input **before tokenizing it**.
  Example:  
  ```
  export HELLO="ho hello"  
  ec$HELLO  
  ```
  The output should be `hello` instead of `ec$HELLO command not found`  
    
1.2 Cut input string into tokens
  This step could be quite annoying, expecially when you realise that the "tokens" in the command line are not necessarily separated by white spaces (or anything).  
  For example, `echo -n message>file|cat file` works just as fine as `echo -n message > file | cat file`.
  What's more, white spaces in quotes should be kept. So we can't simply split the input using ft_split().
