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
