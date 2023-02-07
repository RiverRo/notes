# Unix

> [Main Table of Contents](../README.md)

## In This Notebook

- Unix
- Shell
  - Z Shell
  - Bash
- Add git status to bash command prompt
- General Notes
- A note regarding paths
- Shell constructs
- Shell scripts
- Commands

## Unix

- Unix is an operating system (manages resources, file directories, i/o)

## Shell

- Interfaces between user and the kernel
- Shell is a text-only interface program on the user end
- It is a command-line or command language interpreter
- A script (sequence of commands) that is passed to a shell is a shell script
- A shell script can be executed in any shell
- Shells may be one of Korn, Cshell, Bource, Bash, Z shell (zsh), etc

### Z Shell

- Almost equivalent to Bash but with additional features
- Has a very large collections of plugins
- Some additional features:
  - autocorrect
  - autocompletion
- Is now the default shell for MacOS

### Bash

- Bash is a type of shell and can read bash scripts and shell script
- Bash script is a type of shell script
- Bash has more features than shell
  - 1D arrays
  - Invoked by single or multi-character-command-line options

## Add git status to bash command prompt
- Open "~/.bashrc"
```bash
REPLACE THIS:
if [ "$color_prompt" = yes ]; then 
PS1= ...
else 
    PS1= ...
fi 

WITH THIS:
# git branch info if present
parse_git_branch() {
    git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
}
if [ "$color_prompt" = yes ]; then
   PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[33m\]$(parse_git_branch)\[\033[00m\]\$ '
else
   PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w$(parse_git_branch)\$ '
fi
```

## General Note

- Always use quotes around a filename that contains spaces

## A note regarding paths

- Absolute path values are always the same no matter where I am in the directory
  - Absolute paths start at home
- Relative path values are relative to where I am in the directory
- Root is the name of the head of the entire heirarchical namespace for any UNIX based system
- Home is the head of a user's file directory and is traditionally the directory that the OS places user just after login

## Commands

| Command       | Useful/Common Flags       | Description                                                                                                                                                                                                                                                                                        |
| ------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| $VARNAME      |                           | Access shell variables                                                                                                                                                                                                                                                                             |
| \|            |                           | Unidirectional data channel                                                                                                                                                                                                                                                                        |
| \>            |                           | Redirect<br>e.g. Redirect home path to given file, _overwrite_ if exists: $HOME > log.log                                                                                                                                                                                                          |
| \>\>          |                           | Redirect<br>e.g. Redirect home path to given file, _append_ if exists e.g. $HOME >> log.log                                                                                                                                                                                                        |
| $@            |                           | collection of args passed to a shell script                                                                                                                                                                                                                                                        |
| $#            |                           | Access specific arg passed into a shell script<br>Index position starts with 1                                                                                                                                                                                                                     |
| man [command] |                           | Returns command manual                                                                                                                                                                                                                                                                             |
| head          | -n #                      | Print first 10 lines by default of file or stdin<br>-n Specifiy number of lines                                                                                                                                                                                                                    |
| tail          | -n #                      | Print last 10 lines by default of file or stdin<br>-n Specifiy number of lines                                                                                                                                                                                                                     |
| echo          |                           | Print string(s) to stdout<br>terminal stdin args are strings<br>env vars are strings                                                                                                                                                                                                               |
| cat           |                           | Concatenate file(s) and print to stdout<br>cat: Read stdin<br>cat: when no file and stdin is terminal start interactive console<br>cat > filename.extension: Starts interactive console and writes to give file                                                                                    |
| cut           | <br>-d delimiter<br>-f #  | Print selected parts of file(s) to stdout<br>delimiter to split on<br>after delimiter splits, access the parts/columns, starts at 1<br>e.g. Print third column in csv file to stdout: cut -d , -f 3 filename.ext                                                                                   |
| paste         |                           | Merge lines of file(s)<br>Make sure length of data matches in each file                                                                                                                                                                                                                            |
| grep          | <br>-v pattern<br>-c <br> | Search for pattern and print to stdout each line that contains the pattern<br>inverted search. Match all lines don't contain str<br>suppress normal output. Print count of matching lines for file(s)<br>e.g. Print number of lines not containing pattern 'hello': grep -v 'hello' -c filename.py |
| wc            | <br>-l<br>-w<br>-c<br>-m  | Print number to stdout<br>line count<br>word count<br>byte count<br>char count for file(s)                                                                                                                                                                                                         |
| sort          | <br>-h<br>-r<br>-o        | Sort lines of a file(s) and print to stdout<br>sort by human readable numerical values (e.g. 2K 1G)<br>sort in desc<br>output to a file instead of stdout                                                                                                                                          |
| uniq          | <br><br>-c                | Filter _ADJACENT_ matching lines of file(s) and print to stdout<br>Sort first <br>prefix line with number of occurences                                                                                                                                                                            |

## Shell constructs

| construct     | Description                                 |
| ------------- | ------------------------------------------- |
| VARNAME=VALUE | Create shell variable (local variable)      |
| \$VARNAME     | Access shell variable                       |
| Loop          | for `var` in `iterable`; do `command`; done |

```python
# LOOP
# Remember the shell var needs $ prefix. (ignore quotations)
'for filetype in gif jpg png; do echo $filetype;'

# Do multiple things in one loop
'for file in $files;'
'for file in $files; do head -n 2 $file | tail -n 1; done
```

```python
# LOOP IN SHELL SCRIPT
# Print the first and last data records of each file. (ignore quotations)
'''for filename in $@
do
    head -n 1 $filename
    tail -n 1 $filename
done'''

```

## Shell scripts

- Save shell commands
- Run a shell script with: bash `scriptname`
- Scripts can be used in pipes with other commands
- See `Shell constructs` section for loops in shell scripts

```python
# PASS ARGS TO SCRIPT
# Shell script. (ignore quotations)
'head -n $1 $2 | tail -n 1 | cut -d , -f $3'

# Run script
'bash test.sh 7 somefile.log'  2   # $1=7, $2=somefile.log, $3=2
```
