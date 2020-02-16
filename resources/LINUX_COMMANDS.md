# Linux Commands

* ```ll```: alias for ```ls -l```. This command is used to get detail information about files and directories in present working directory.
* ```pkill```: The *kill* command is a very simple wrapper to the *kill* system call, which knows only about process IDs (PIDs). *pkill* and *killall* are also wrappers to the *kill* system call, (actually, to the libc library which directly invokes the system call), but can determine the PIDs for you, based on things like, process name, owner of the process, session id, etc.
* ```mkdir -p /data/consul/{data,config,ui}```: can use ```{}``` braces to create mutiple directores in one command!
* ```tee```: read from standard input and write to standard output and files e.g. ```cat /vagrant/ansible/roles/haproxy/files/haproxy.cfg.orig *.service.cfg | tee haproxy.cfg```
