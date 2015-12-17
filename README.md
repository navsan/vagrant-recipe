# Vagrant + GPDB + Postgres
Steps to set up a dev environment for GPDB and Postgres using Vagrant


## Get a Vagrant box running
* Initialize vagrant (say in `~/vagrant-centos`)
```bash
vagrant init
```
* Choose a vagrant box from the [Vagrant cloud](https://vagrantcloud.com/discover/featured) website and add it
```bash
vagrant box add puppetlabs/centos-7.0-64-puppet
```
* You may have to choose a provider. I'm going with virtualbox.
* In the Vagrantfile (created during init), set
```bash
config.vm.box = "puppetlabs/centos-7.0-64-puppet"
```
* You may also want to configure the memory and cpu settings in the file (before `end` in the file)
```bash
config.vm.provider "virtualbox" do |v|
  	v.memory = 4096
  	v.cpus = 3
end
```
* I had to set up virtualbox guest additions plugin in vagrant. Without this plugin, the vagrant box wouldn't boot up the *second* time around. See [this](https://github.com/dotless-de/vagrant-vbguest).
```bash
vagrant plugin install vagrant-vbguest
```
* Now start vagrant
```bash
vagrant up
vagrant ssh
```
* For some reason that I can't really figure out, rebooting the machine using `vagrant up` goes through a connection timeout-retry cycle 3 times. It eventually boots up after about 30s, so this is only a minor annoyance, not a major hurdle. I'm going to leave it as is (after burning an hour trying to fix it). This doesn't affect `vagrant ssh`. Try using another vagrant base box if you're really bothered. Or changing the BIOS settings. Or changing some networking settings. Or changing the authentication method.

## Install development prerequisites
After ssh-ing into the vagrant box
```bash
cd ~
sudo yum -y update
sudo yum -y groupinstall "Development tools"
sudo yum -y install ed readline-devel zlib-devel curl-devel   \
		bzip2-devel python-devel epel-release htop
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
sudo pip install psi lockfile paramiko setuptools epydoc
rm get-pip.py
```

## [Optional] Upgrade to Python 2.7
I prefer to have a modern version of Python for convenience.
```bash
sudo yum install -y zlib-dev openssl-devel sqlite-devel bzip2-devel
cd ~/Programming
wget https://www.python.org/ftp/python/2.7.11/Python-2.7.11.tgz
tar zxvf Python-2.7.11.tgz
cd Python-2.7.11
./configure  --enable-shared
make -j 4
sudo make install

```
Note that I had to restart the shell session for the new python to be reflected.


## [Optional] Install and configure zsh
* I prefer using zsh as my default shell
```bash
sudo yum -y install zsh
zsh   ## Choose 0 to create .zshrc
exit
```
* To change the default shell using chsh, you need the password of the current user (vagrant). For the box I'm using, it's not a standard password, so I reset it to vagrant by running
```bash
sudo su
passwd vagrant
```
* oh-my-zsh makes zsh a delight to work with
```bash
cd ~
bash -c "$(wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```
Note that the oh-my-zsh installation script asks for your password to change your default shell to zsh. Use the password you set above (vagrant).
* I like to keep all my shell configuration settings in one common `~/.myrc` file that's shared between bash and zsh
```bash
echo "\n\n## ~/.bashrc is marked READ-ONLY. Use ~/.myrc instead." >> ~/.bashrc
echo "if [ -e ~/.myrc ]; then source ~/.myrc; fi" >> ~/.bashrc
chmod a-w ~/.bashrc
echo "\n\n## ~/.zshrc is marked READ-ONLY. Use ~/.myrc instead." >> ~/.bashrc
echo "if [ -e ~/.myrc ]; then source ~/.myrc; fi" >> ~/.zshrc
chmod a-w ~/.zshrc
touch ~/.myrc
```


## [Optional] Install and confiugre vim
* I usually compile vim from source to enable the features I want
```bash
mkdir -p ~/Programming
cd ~/Programming
git clone https://github.com/vim/vim.git
cd vim
sudo yum -y install cscope cmake automake
PY_CONFIG=`echo "import sysconfig as s; print s.get_config_var('LIBPL')" | python`
./configure --enable-cscope --with-features=huge --enable-pythoninterp --with-python-config-dir=$PY_CONFIG
make -j 4
sudo make install
sudo mv `which vi` `which vi`.bkp
sudo ln -s `which vim` `dirname \`which vim\` `/vi
sudo chmod +x `which vi`
```
You may need to start a new shell session after the above. Check `vi --version` to ensure it's the one that you just compiled, and that it has python and cscope support turned on.
* I manage all my vim plugins using Vundle
```bash
cd ~/Programming/vim
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```
* My favorite color scheme is wasabi256
```bash
cd ~/Programming/vim
git clone https://github.com/thomd/vim-wasabi-colorscheme.git
cd vim-wasabi-colorscheme/
mkdir -p ~/.vim/colors
make
```
* My `.vimrc` settings are available in a bitbucket repo.
```bash
cd ~
git clone https://navsan@bitbucket.org/navsan/dotfiles.git
ln -s ~/dotfiles/.vimrc ~/.vimrc
ex -c ":PluginInstall"
```
That last command installs all the Vundle plugins I use.
* I use [YouCompleteMe](https://github.com/Valloric/YouCompleteMe)  for semantic code completion
```bash
cd ~/.vim/bundle/YouCompleteMe
./install.py --clang-completer
```

## Setting up GPDB
* Download the sources and build it
```bash
mkdir -p ~/Programming
cd ~/Programming
git clone https://github.com/greenplum-db/gpdb.git
cd gpdb
mkdir -p gpdb-dist
./configure --prefix=$PWD/gpdb-dist --enable-depend --enable-debug
make -j 4
make install
```
* Apply the configuration settings by adding this to `~/.myrc` (or `~/.bashrc`)
```bash
# GPDB settings
export GPDB=~/Programming/gpdb
export GPHOME=$GPDB/gpdb-dist
export GPDATA=~/gpdata
```
You could source `greenplum_path.sh` in your `~/.myrc` (or `~/.bashrc`) file, but I prefer not to do so. It messes up the PYTHONPATH and LD_LIBRARY_PATH. If you don't mind that, then add this to `~/.myrc`.
```bash
if [ -e $GPHOME/greenplum_path.sh ]; then
		source $GPHOME/greenplum_path.sh
fi
```
Or else you can add this instead. Caveat emptor: it seems to work so far, but could lead to problems later.
```bash
export OPENSSL_CONF=$GPHOME/etc/openssl.cnf
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GPHOME/lib:$GPHOME/ext/python/lib
```
* Source the above script
```bash
source ~/.myrc
```
* Create the data directories
```bash
mkdir -p $GPDATA/master
mkdir -p $GPDATA/segments
```
* Set up ssh keys and hostfile
```bash
gpssh-exkeys -h `hostname`
hostname >> $GPDATA/hosts
```
After this, you should be able to successfully log in using the following command, without having to enter a password.
```bash
ssh `hostname`
```
* Create the gpinitsystem config file. You can use the minimal file created by the commands below or you can make the appropriate changes in the `$GPHOME/docs/cli_help/doc/gpconfigs/gpinitsystem_singlenode` file. We are setting up a "cluster" with one master and two segments, all within this vagrant box.
```bash
GPCFG=$GPDATA/gpinitsystem_config
rm -f $GPCFG
echo "declare -a DATA_DIRECTORY=\
	($GPDATA/segments $GPDATA/segments)"       >> $GPCFG
echo "MASTER_HOSTNAME=`hostname`"            >> $GPCFG
echo "MACHINE_LIST_FILE=$GPDATA/hosts"       >> $GPCFG
echo "MASTER_DIRECTORY=$GPDATA/master"       >> $GPCFG
echo "ARRAY_NAME=\"GPDB\" "                  >> $GPCFG
echo "SEG_PREFIX=gpseg"                      >> $GPCFG
echo "PORT_BASE=40000"                       >> $GPCFG
echo "MASTER_PORT=5432"                      >> $GPCFG
echo "TRUSTED_SHELL=ssh"                     >> $GPCFG
echo "CHECK_POINT_SEGMENTS=8"                >> $GPCFG
echo "ENCODING=UNICODE"                      >> $GPCFG
```
* Configure OS settings by adding these lines in `/etc/sysctl.conf` as per instructions [here](http://gpdb.docs.pivotal.io/4360/prep_os-system-params.html#topic3). The security limits and other settings don't seem to be necessary.
```
kernel.shmmax = 500000000
kernel.shmmni = 4096
kernel.shmall = 4000000000
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ip_local_port_range = 1025 65535
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.overcommit_memory = 2
```
Following this, you must **reboot** the vagrant box.
```bash
exit    # from the vagrant box ssh shell
vagrant halt
vagrant up
vagrant ssh
```
* Initialize the database with the config file we created above
```bash
source $GPHOME/greenplum_path.sh
$GPHOME/bin/gpinitsystem -a -c $GPDATA/gpinitsystem_config
```
In case of any errors, look in the log files at `~/gpAdminLogs`.
* If the initialization ran successfully, the output would tell you to add
```bash
export MASTER_DATA_DIRECTORY=$GPDATA/master/gpseg-1
```
to your `~/.myrc`. Doing so will mean you don't have to specify the master data directory location every time you start/stop the server or client. On the other hand, if you wish to work with multiple data instances or with both Postgres as well as GPDB, adding this `export` command will likely make things very confusing.
* You can now use `gpstate` to view the state of the cluster. You can also use `gpstart` and `gpstop` to start and stop the database server.
* Now create a database and use the client to connect to it. If you use the user name as the DB name (as in the commands below), you don't have to specify the DB name in the `psql` command. You can also set the environment variable `DATABASE_NAME` to auto-connect to it. I usually set it to `tpch` and load the benchmark data there.
```bash
DBNAME=$USER
createdb $DBNAME
psql $DBNAME
```

## Setting up Postgres
* Download and build Postgres source code
```bash
mkdir -p ~/Programming
cd ~/Programming
git clone https://github.com/postgres/postgres.git
cd ~/Programming/postgres
./configure --enable-debug --enable-depend --prefix=`pwd`/pg-dist
make -j 4
make install
```
* Add the following lines to `~/.myrc` (or `~/.bashrc`) and source it.
```bash
export PGDB=~/Programming/postgres
export PGHOME=$PGDB/pg-dist
export PGDATA=~/pgdata
```
* Initialize the database
```bash
cd ~
mkdir -p $PGDATA
$PGHOME/bin/initdb $PGDATA
```
* Now you can start and stop the database server using
```bash
$PGHOME/bin/pg_ctl -D $PGDATA -l $PGDATA/db.log start
$PGHOME/bin/pg_ctl -D $PGDATA -l $PGDATA/db.log stop
```
* You can now connect to the database using the following client command.
```bash
$PGHOME/bin/psql
```
