# Rhea-Public
 
DevotedMC 1.17.1 Ansible sharding deployment environment


## How to deploy:

First of all note that Ansible does not support Windows, you will need a UNIX based operating system. DevotedMC uses Debian and all instructions in the following will be based on that (should also work on Ubuntu). Note that the Windows Subsystem for Linux is NOT a UNIX based operating system, do not report issues when trying to run this setup on it and install a proper OS.


- Install Java (minimum 11)

https://adoptopenjdk.net/

- Install Git, Python and MariaDB

`sudo apt-get install git python python3-pip python-dev default-libmysqlclient-dev python3-pymysql mariadb-server mariadb-client byobu curlftpfs dirmngr rabbitmq-server`

If `python3-mysql` can't be found or you are later on getting an error like ` "The PyMySQL (Python 2.7 and Python 3.X) or MySQL-python (Python 2.X) module is required."` try `sudo apt-get install python-pymysql`

Recommendend optionals for managing a server:

`sudo apt-get install htop iotop`

- Install Ansible

https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

- Configure a root password for MariaDB

https://www.digitalocean.com/community/tutorials/how-to-reset-your-mysql-or-mariadb-root-password

Note that in the following the terms MariaDB and Mysql are used interchangeably

- Create a user under which the server will be running

By default this user is named mc, so: `sudo adduser mc`
You may use a different user name, if you do so you will need to adjust the user name setting in `variables/all.yml`


- Clone this repo
```
git clone https://github.com/DevotedMC/Rhea.git
cd Rhea
```

- Create the file variables/passwords.yml and fill it as follows:

```
#Name of the mysql root user
mysql_root_user: root
#Mysql root password
mysql_root_pass: squidlover69
#Host of the mysql database
mysql_host: localhost
#Port mysql is using, 3306 by default
mysql_port: 3306
#Seeds used for HiddenOre noise generation
hidden_ore_density_seed: 1234
hidden_ore_height_seed: 1234
#Seed used for FactoryMod wordbank
wordbank_seed: 1234
#Optional API key to make Bansticks ip hub lookup work
ip_hub_key:
#BungeeGuard Key used to avoid BungeeSpoofing (change this if you don't have a firewall setup properly)
bungee_guard_token: 1234
#FTP user name to use for mounting backup point
remote_backup_user: squid
#Password to use for mounting FTP backup point
remote_backup_password: ink
#URL to use for FTP backup point
remote_backup_server: some.backup.domain

mysql_passwords:
  alpha: abcd
  beta: abcd
  gamma: abcd
  delta: abcd
  lobby: abcd
```

You should only have to change `mysql_root_pass` to your MariaDB root password and change the values under `mysql_passwords` to arbitrary random string for this to work

- Configure the setup according to your needs

Check out `variables/all.yml` and change values as you see fit. You will most likely to use less RAM than we do.

- Run the deploy script

`sudo bash server.sh deploy`

This script requires sudo for the initial database setup. You only need to run it as root once at the beginning or if you want to change the database or user name used. Past that any invocations of `server.sh` should not require sudo. `server.sh` is your single access point to controlling your entire setup, run `bash server.sh help` to see a full list of available commands. 

---

For reference, Civclassic has the following cronjobs setup for the user `mc`:

```
44 7 * * * mv /home/mc/restart.log.txt /home/mc/restart.old.log.txt
45 7 * * * cd /home/mc/AnsibleSetup && bash server.sh pull >> /home/mc/restart.log.txt
50 7 * * * cd /home/mc/AnsibleSetup && bash server.sh warn10 stop backup update start >> /home/mc/restart.log.txt
```


## Structure

#### Binaries

Binaries are under `jarfiles` with folders for Zeus, Minecraft and BungeeCord each. In these folders can also be jar files with the file ending `.jar.priv` which are ignored by git, but will still be deployed.

#### Template files

All server specific configuration files are in `templates`. Four folder are provided here, `zeus`, `bungee`, `shards` and `categories`. 
`zeus` and `bungee` follow the same structure with the directory itself containing templates relevant to the application itself, which will be placed in the same folder as the services jar. These top level templates are explicitly listed in `variables/all.yml`. 
Folders within these directories each contain configuration files of a single plugin with the same name as the folder. Configuration folders will be deployed recursively, keeping the provided directory structure intact, relative to the plugins data directory. All configuration files must have the file ending `.j2` or `.j2.priv`. These file ending will be removed during deployment and `.j2.priv` files will overwrite `.j2` files with the same name.

Minecraft servers are subdivided into categories with the ability to define a configuration for an entire category and overwriting category wide configuration files for individual shards. The folder `templates/categories` contains a folder for every minecraft server category. 
Each of these categories is structured the same way as the `zeus` or `bungee` folder, including the mapping to plugins, top level templates, private configuration files and configuration deployment. The folder `templates/shards` contains a folder for each shard. These folders follow the same structure as category folders, but their content will take precedence over that of the minecraft servers category configuration files.

The precedence order for a config file is:

- Public configuration file of the category
- Private configuration file of the category
- Public configuration file of the shard
- Private configuration file of the shard

So for example for a plugin called `Citadel` on the shard `alpha` in the category `nether` will have the following precedence with ones further down the list taking higher priority (if they exist):

- templates/categories/nether/Citadel/config.yml.j2
- templates/categories/nether/Citadel/config.yml.j2.priv
- templates/shards/alpha/Citadel/config.yml.j2
- templates/shards/alpha/Citadel/config.yml.j2.priv


If configs only differ in single values, per shard values can be defined in `variables/all.yml`


#### Variables

All variables are under `variables`. The intended setup is to have all global public configuration settings in `all.yml` and credentials in `passwords.yml`. Further each shard category has a configuration file based on the category name which for example for a category called `nether` would be `nether_config.yml`. Namespaces of variables may not overlap, except for configuration files of different categories. 
Public per shard configuration options can be added in the `shards` list of `all.yml` and be referenced by `shard.config_option_name` in the shards configuration files. 
Secret per shard configuration options can be realized via a section in `passwords.yml` like:

```
mysql_passwords:
  alpha: abcd
  beta: abcd
  gamma: abcd
  delta: abcd
  lobby: abcd
 ```

 with one sub entry for every shard using the config value. If a shard does not use the variable, it does not have to be listed here. In configs these values can be referenced like `'{{ mysql_passwords [shard.name]}}'`
