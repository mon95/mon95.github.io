---
layout: post
title: Building a Bash-like application
---

![The batch job executer in action](/images/bp3pic1.png)

I was recently working on implementing a Batch Job Executer to get familiar with the basics of using **fork()**, **exec()** and its variants, **dup()/dup2()** and **pipe()**. Simply put, a Batch Job Executer is a program that is designed to execute a batch job (i.e., a set of commands given as an input in a text file).

I’ve put together a few pieces of code to build a simple batch job executer by extending some of the work that I had read about while doing it and explained it all here! But before that …

### Nobody is going to use my program! Why should I even build a bash-like application?
(Honest Answer: Somebody else asked me to!)

No, seriously!

One of the major takeaways of completing this task, apart from the fact that I learnt how the above mentioned system calls worked, was that when I was done implementing it, I was able to truly appreciate and understand the kind of work that has gone into providing even the simplest of features on the command line. At the same time, as pointed out by Stephen Brennan in his blog (which I must admit was what helped me to get started on the task), I learnt that it isn’t very hard to code up the basic idea behind a software or a program that everyone uses.

Here’s an extract from his blog post:

> It’s easy to view yourself as “not a real programmer.” There are programs out there that everyone uses, and it’s easy to put their developers on a pedestal. Although developing large software projects isn’t easy, many times the basic idea of that software is quite simple. Implementing it yourself is a fun way to show that you have what it takes to be a real programmer.

In this case, if you know some basic(and I mean very basic) C programming, you can come up with a working version of an application that simulates some of the basic functionalities of a shell!

### What exactly are we designing?
The end product will be a simple batch job executer that takes in an input file and executes single line commands from the file, very similar to how the bash would execute a shell script file. [Here’s a link to the code](https://github.com/mon95/Batch-Job-Executer). (Do check the github page for exact design specifications and consolidated code)

As the main objective is to simply understand, appreciate and use the fork, dup, exec and pipe system calls, we’re only going to be designing a simplified version of the same. Some of the assumptions we are going to make are:
1. Batch file contains only single line commands. (i.e., commands are not extended to multiple lines using ‘\’)
2. Built-in shell commands are not used. (Can be easily incorporated, but doesn’t really contribute to understanding the system calls listed above)
3. Only single line comments using ‘#’ preceded and followed by a space are handled. (i.e., Handling of multi line comments using /* .. */ is not supported)
4. Only commands between a %BEGIN and a %END are executed. All other commands are ignored.
5. We redirect output to a specific file (here, output.txt). (Note that by default, output is directed to stdout)
6. Commands handled only include the ‘|’, ‘>’ and ‘>>’ operators (i.e., operators like ‘&’ and ‘<’ are not supported. But, the same code can be easily extended to do the same)

With the above assumptions, it’s easy to focus on just doing the important stuff! (For, example, the whole thing can be made quite sophisticated using a better parser or with a little extra code we can handle built-in commands. But we don’t really have to do all of that now!)

### The basic outline
We’ll need:
1. A function that parses the input line (i.e., the command)
2. Function(s) that help with the execution of the parsed commands
3. A driver function (i.e., the main function)

### The parsing:
Let’s define a header file called ‘parse.h’ as follows:

```
#define tok_delimiters " \t\n\r"
#define tok_size 128
/*
    Function to parse a space separated line containing a command and its corresponding args.
*/
char** parseLine(char *line);
```

Then, define the parseLine() function as follows in a file called ‘parse.c’:

```
char** parseLine(char *line) {
    int len = strlen(line);
    int argsize = tok_size;
    char** args = (char**)malloc(argsize * sizeof(char*));
    int pos = 0;
    char* arg;
    if(!args) {
        // Not enough space available. Return failure and exit.
        printf("Memory allocation unsuccessful! Exiting...\n");
        exit(EXIT_FAILURE);     
    }
    /*
    * Key parsing logic:
    * Get the first argument. 
    * While the argument is valid (i.e., not null), 
    * add it to the  list of arguments that will be 
    * returned to the main function.
    * Take care of running out of space using realloc.
    */
arg = strtok(line, tok_delimiters);
    
    while(arg != NULL) {
        // Don't parse after the beginning 
        // of a single-line comment is detected   
        if(strcmp(arg, "#") == 0) break;    
        
        args[pos] = arg;
        pos = pos + 1;
        if(pos > argsize) {
            // reallocate memory for args
            argsize = argsize + tok_size;
            args = realloc(args, argsize*sizeof(char*));
            if(!args) {
              // Not enough space available. 
              // Return failure and exit.
              printf("Memory allocation unsuccessful! Exiting!\n");
              exit(EXIT_FAILURE);     
            }
        }
       arg = strtok(NULL, tok_delimiters);
    }
    
    args[pos] = NULL;
    return args;
}
```

Here, we use [strtok()](http://stackoverflow.com/questions/21097253/how-does-the-strtok-function-in-c-work?noredirect=1&lq=1) to tokenize the input. Here, we parse the command (including the ‘|’, ‘>’ or ‘<’ operators, if any) to return an array of args, which is later used to execute them.

### The execution of the parsed commands:
We’ll need to define a couple of functions that handle the essential execution logic. For this, include a header file called ‘ execute.h’ as follows:

```
// Finds the length of the array of arguments
int argsLength(char **args);
/*
Main logic to perform the necessary tasks.
Input is an array of arguments.
This function only handles "|", ">" and ">>" operators.
By default, all final output is redirected to a file-'OUTPUT.txt'.
*/
int execute(char** args);
/*
Function that handles multiple pipes.
    commands[]      :   Array of commands 
    numberOfCommands:   number of commands in commands[]
    op1             :   pos of '>' ; indicates presence of '>'
    op2             :   pos of '>>' ; indicates presence of '>>'
    redirectfile    :   filename to be used in case of redirection
*/
void executePipeCommands(char** commands[], int numberOfCommands, int op1, int op2, char* redirectfile);
```

Here, for correct parsing, all input in the batch file is assumed to have correct (non built-in functions only) input, separated by spaces.

Execute.c:

Understanding how the execute function works:

* Get the length of the arguments array (i.e., no of arguments)

```
// Defined in execute.c
// Returns the length of the array of arguments 
int argsLength(char **args) {
  int i = 0;
  while(args[i] != NULL) {
    i++;
  }
  return i;
}
```

* Iterate and get the positions(if present) of the ‘>’, ‘>>’ and the ‘|’ operators (Irrelevant note: this implementation returns the position of the last ‘|’ found)

```
int execute(char** args) {
pid_t pid, wpid;
int status;
int len = 0;    // length of the argument list
int i = 0;   
int op1 = 0;     // to mark pos of ">"
int op2 = 0;     // to mark pos of ">>"
int opPipe = 0;  // to mark pos of "|"
int out_fd;
int out;
len = argsLength(args);
for(i=0;i<len;i++) {
  if(strcmp(args[i], ">") == 0) { op1=i; }         // pos of ">"
  else if(strcmp(args[i], ">>") == 0) { op2=i; }   // pos of ">>"
  else if(strcmp(args[i], "|") == 0) { opPipe=i; } // pos of "|"
}
// ... 
```

* Handle on a case by case basis:
Case 1: No |, > or >> operator. Redirect output to OUTPUT.txt

```
// ...
// Case: No |, > or >> operator. Redirect output to OUTPUT.txt
if(op1 == 0 && op2 == 0 && opPipe == 0) {
  if((pid = fork()) < 0) 
    perror("Fork error");
  else if(pid == 0) {
    // Child. 
    // Execute command here
    int fd = open("OUTPUT.txt", O_RDWR | O_CREAT | O_APPEND, S_IRUSR | S_IRGRP | S_IWGRP |S_IWUSR); 
    dup2(fd, 1);    // to make stdout go to file. 
    dup2(fd, 2);    // to make stderr go to file. 
    close(fd);
    printf("\n\n");   
    execvp(args[0], args);
    printf("Couldn't execute this command\n");
  }
  else {
        // wait for the child in parent process
        do {
          wpid = waitpid(pid, &status, 0);
        } while(!WIFEXITED(status) && !WIFSIGNALED(status));
  }
}
// ... 
```

This is the simplest of the three cases.

We use the fork() system call to fork a child process and this child process is then used to execute the command.

For using fork() in a context like this, generally, your code always looks like this:

```
// Using fork() 
if ((pid = fork()) < 0) // fork error
  perror("Fork error");
else if (pid == 0) // child process.
{ 
  // do child process stuff
}
else // parent process
{
  // do parent process stuff. 
  // in our case this only involves waiting for child to finish
}
```

To execute commands we have a set of system calls-execl(), execlp(), execle(), execv(), execvp(), execvpe().

In our case, we have already parsed our command into an array of arguments, where, the first argument is the name of the command. To execute this, we’ll need to use execvp(). Here, the first argument is the file name(i.e., the name of the command-args[0]) and the second is a null terminated array of strings. **The exec() family of functions replaces the current process image with a new process image**. This means that any code after the execvp() function will not be executed.

Now, for the last part, i.e., to redirect the output to go to a file, we use the **dup2()** system call.

The open() system call assigns the file descriptor of ‘OUTPUT.txt’ to the variable fd. Then, dup2(fd, 1) simply makes 1(which is the fd for stdout) be the copy of fd, closing stdout first if necessary. Think of the command as simply replacing ‘stdout ’with ‘OUTPUT.txt’.
    
Now, on execution, the outputs are written to OUTPUT.txt!

The parent process simply waits for the child process to finish execution.

Case 2: ‘>’ operator found or ‘>>’ found.

The only difference here is that we have to modify our output file descriptor accordingly.

```
if (op1 != 0) {
  // '>' operator
  args[op1] = NULL; // arguments must be null-terminated   
  // open in truncate mode - O_TRUNC    
  out_fd = open(args[op1 + 1], O_WRONLY | O_TRUNC | O_CREAT, S_IRUSR | S_IRGRP | S_IWGRP |S_IWUSR);
 }
else {
  // '>>' operator
  args[op2] = NULL; 
  // open in append mode - O_APPEND
  out_fd = open(args[op2 + 1], O_CREAT | O_APPEND | O_WRONLY, S_IRUSR | S_IRGRP | S_IWGRP | S_IWUSR);
 }
// rest is the same as previous case
// replace stdout with this out_fd
   dup2(out_fd, 1);  // 1 equivalent to STDOUT_FILENO
   close(out_fd);
   printf("\n\n");
   execvp(args[0], args); 
}
```

Case 3: ‘|’ operator is present.

For this particular case, we need to parse and extract the individual commands that are being piped and store them in a separate array.

```
int n = numberOfPipes(args);
char **commands[n+1];   // Array of cmds after splitting on '|'
```

The parsing is pretty straightforward and is done as follows:

```
// Parse the commands and add them to the commands array
while (k<n+1) {
  commands[k] = (char**)malloc(len * sizeof(char*));
  if(!commands[k]) exit(EXIT_FAILURE);
     j = 0;
     while(curoffset < len && strcmp(args[curoffset],"|") != 0 && strcmp(args[curoffset], ">") != 0 && strcmp(args[curoffset], ">>") != 0) {
         commands[k][j] = args[curoffset];
         curoffset = curoffset + 1;
         j = j + 1;
     }
          
     commands[k][j] = NULL;
     k = k + 1;
     curoffset = curoffset + 1;
}
```

We then invoke a function, executePipeCommands() for the final execution.

```
void executePipeCommands(char **commands[], int n, int op1, int op2, char *redirectfile) {
int fin, fout;
fin = dup(0)
// ...
// Iterate over all commands
for(i=0; i<n; i++) {
dup2(fin, 0);
close(fin);
if(i == n-1) {
  // If it's the last command, 
  // check where the o/p is to be redirected
  if(op1 != 0) {
    fout = open(redirectfile, O_WRONLY | O_TRUNC | O_CREAT, S_IRUSR | S_IRGRP | S_IWGRP | S_IWUSR);
  }
  else if(op2 != 0) {
    fout = open(redirectfile, O_WRONLY | O_APPEND | O_CREAT, S_IRUSR | S_IRGRP | S_IWGRP | S_IWUSR);
  }
  
  else {
  // No redirectio. So, use OUTPUT.txt
    fout = open("OUTPUT.txt", O_RDWR | O_CREAT | O_APPEND, S_IRUSR | S_IRGRP | S_IWGRP |S_IWUSR); 
  }
}
else {
      // Use pipe for everything in between. 
      int pipefd[2];
      pipe(pipefd);
      fin = pipefd[0];
      fout = pipefd[1];
}
dup2(fout, 1);
close(fout);
if((pid = fork()) < 0) 
      perror("Fork error");
else if(pid == 0) {
      // child
      execvp(commands[i][0], commands[i]);
      printf("Couldn't execute this command\n");
    }
else {
      // parent
      do {
          wpid = waitpid(pid, &status, 0);
        } while(!WIFEXITED(status) && !WIFSIGNALED(status));
}
```

This execution part is the same as the previous cases. The difference is in the usage of the pipe() system call to redirect the input and output. pipefd[0] is the read end of the pipe and pipefd[1] is the write end.

The dup2(fin, 0) ensures that fin replaces the stdin file descriptor. (Initially, we set fin=dup(0), i.e., stdin)

Thus, what happens is, all the commands in between two pipes (i.e., ‘|’) read the input from the read-end of the pipe and write to the write-end.

### The driver function:
The driver function looks something like this:

```
int main(int argc, char **argv) { 
  // ...
  if(argc != 2) {
    printf("Usage: ./executeBatchJobs <file-to-be-executed>\n");
    return 0;
  }
  
  // Readying the OUTPUT.txt using O_TRUNC
  int fd = open("OUTPUT.txt", O_RDWR | O_CREAT | O_TRUNC, S_IRUSR | S_IRGRP | S_IWGRP |S_IWUSR); // user - r and w permissions. 
  close(fd);
  
  fp = fopen(argv[1], "r");
  while(getline(&line, &linesize, fp) != -1) { 
    // ...
    // If line is within a begin-end block, then, parse and execute.
    // Else, ignore the line.
    // ...
  }
  // ...
}
```

All the above files(i.e. the complete code) can be found on [my github](https://github.com/mon95/Batch-Job-Executer).

*The above code could have been much much better. For instance, there isn’t really a need to have a separate function handling pipes. One function, with simpler code, would have been nicer.*

*I’d done this as part of an assignment and I was only trying to get things to work. And didn’t really end up doing any refactoring. Anyway, the point was to demonstrate that it’s totally possible to get pretty cool stuff done using simple things we already know about while also learning how to use a few common system calls!*

*Cheers!*


(Links that I referred to: [Stephen Brennan’s Write a Shell in C](https://brennan.io/2015/01/16/write-a-shell-in-c/), [CS702 Lecture Notes (Dr. James Glenn)](http://www.cs.loyola.edu/~jglenn/702/S2005/Examples/dup2.html) and [aeb5005’s github gist](https://gist.github.com/aeb5005/16d0a61608918a6a3567))
