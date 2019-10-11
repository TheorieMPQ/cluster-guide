# Configuring

You may notice that packages in `TheorieMPQ` are private and that you must be authenticated to download them.

To avoid having to type in the password everytime you install a package or you update them, follow the following instructions.

## Generating a ssh key and adding it to GitHub

First we need to generate an ssh key if you don't have one yet. 
Copy and paste this command in your terminal.
```shell
if test -f "~/.ssh/id_rsa"; then echo "Key alread exists. Do nothing."; else ssh-keygen -f ~/.ssh/id_rsa -q -N; fi
```

Now you surely have an ssh key in `~/.ssh/id_rsa`, and we must upload it to github.
To do that, you can follow [this guide](https://help.github.com/en/enterprise/2.15/user/articles/adding-a-new-ssh-key-to-your-github-account) or 
the instructions below.

- copy the content of the file `~/.ssh/id_rsa.pub`. To do so, run the command and copy all the output
```shell
clear && cat ~/.ssh/id_rsa.pub
```
- go to [this page on GitHub](https://github.com/settings/keys)
- click on *New SSH Key*
- paste the content of `~/.ssh/id_rsa.pub`
- click *Add SSH Key*

## Set up automatic login

To enable automatic logging you should add a few lines to your `.ssh/config` file. 
To do so, run the command
```shell
nano ~/.ssh/config
```

and paste the following lines
```
Host github.com
   HostName github.com
   User git
   IdentityFile ~/.ssh/id_rsa
```

You're done! To check that everything worked you can run
```shell
ssh github.com
```
and you should see the following output
```
Hi YOURUSERNAME! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

## Adding Packages to Julia

From now on, when you add private packages to Julia, if they are of the form
```
] add git@github.com:TheorieMPQ/CornerSpaceRenorm.jl.git
```
you will not be asked for a password.
