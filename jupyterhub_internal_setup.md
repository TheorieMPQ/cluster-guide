# Jupyterhub internal setup

This is a short summary of how jupyterhub is setup up on jupyterhub. In case anyone ever needs to edit it.

## Where is it installed 

- JupyterHub is installed in `/opt/jupyterhub` of thtop. 
- Important files are `/opt/jupyterhub/jupyterhub_config.py` which contains all settings.
- Logs are stored in `/opt/jupyterhub/rundir/`, and they are `out.log` and `err.log` for stdout and errors. You should start from here to understand what's broken
- The conda environment for jupyterhub (and only the hub, not users) is located at `/opt/jupyterhub/jupyterhub-host-env`

I installed jupyterhub as a service, so it should startup when thtop is turned on.
The service file, that should be used to run it (JUPYTERHUB SHOULD NEVER BE LAUNCHED MANUALLY) is located at 
`/etc/rc.d/init.d/jupyterhub`.

To launch/stop/restart jupyterhub you can use the following three commands (as sudo)
```
service jupyterhub restart
service jupyterhub stop
service jupyterhub start
```

At every restart log files are deleted.

## Environment
Jupyterhub is installed as a conda environment that can only be modified by super users.
To access this environment you can run as sudo user (sudo su) 
 - `module load anaconda`
 - `source activate /opt/jupyterhub/jupyterhub-host-env`

This is necessary for example to update jupyterhub or the jupyterlab version, or to install jupyterlab plugins.
Be cautious if you upgrade! In particular I hand-edited two packages, `pbsspawner` and `sshspawner`, so if you upgrade them they won't probably work anymore.
they are installed through pip, so if you upgrade through conda everthing should be fine, but... who knows.


## Editing configurations
`/opt/jupyterhub/jupyterhub_config.py` contains a list of profiles that can be spawned. Search for `c.ProfilesSpawner.profiles` in it.
The documentation for this stuff is in the ProfilesSpawner github repo.

If you want to add new configurations you should add them here and then restart jupyterhub.

## Using SSH spanwer (launch jupyter clients on other computers like th0)
This is very brittle, so use with caution.
This spawner does not set any limit on th0 usage, so people might use all computing power and starve the others...
Moreover, th0 and so on have a different filesystem from th-top nodes, and it's older, and it might have many other problems...

to make it work, every user must (by himself) ensure the following things:
 - on th0 he must have an account with *exactly the same username* than his username on th-top
 - he must have enabled passwordless key authentication from thtop to th0 ([see here](https://linuxize.com/post/how-to-setup-passwordless-ssh-login/)). Moreover the ssh key on thtop should be located exactly at `/home/$USER/.ssh/id_rsa`. wonb't work othgerwise.
 - he must install by himself, for himself his jupyter kernels.
 
 
Assuming an user has a good user account on th0, the way to setup passwordless is the following:
 - check if there is a file in `~/.ssh/id_rsa` 
 - if there is none, do the following: `ssh-keygen -t rsa -b 4096 -C "$USER@master0@th-top.mpq.univ-paris-diderot.fr"`
 - when he asks for a path, just press enter so he defaults to `~/.ssh/id_rsa` (check that this is the default)
 - when he asks for a password, don't set any (press enter)
 - run `ssh-copy-id $USER@th0.mpq.univ-paris-diderot.fr`
 
from now on `th0` should work for the user.
