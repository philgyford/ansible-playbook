# My first Ansible playbook

Very in progress.

Designed to host multiple websites, from different git repositories, on a single webserver. Can be used with a Vagrant virtual machine and a DigitalOcean droplet (not tested with anything else).


## Apps

You'll need to alter `roles/apps/vars/main.yml` to reflect the websites (called "apps" here you want to install). Each item in the list has one or more of these elements:

* `name`: Required. No spaces. Will be used as a directory name for where its git repo will be checked out to. A single directory name that will be put within `apps_path` (defined in an `env_vars/*.yml` file). Also used for a virtualenv name, if required.

* `git_repo`: Optional. The `https://github.com...` path to the git repository. It won't work with a `git@github.com...` path. Although optional, not much will happen without this.

* `git_repo_branch`: Optional. The name of the branch to check out. If omitted, defaults to `'master'`.

* `python_version`: Optional. e.g. `3.5.1`. If set, will be used to create a python virtualenv for this app using pyenv.

* `pip_requirements_file`: Optional. If the app has a pip requirements file, set the path to it within the repo here. eg, `requirements.txt`. Otherwise, omit this. If set, the python packages will be installed.

* `django_settings_file`: Optional. If this is a Django site, this is the path to the settings file to use within the repo. eg `myapp/settings/production.py`. If this is defined then the Django `syncdb`, `migrate` and `collectstatic` management commands will be run.


## Vagrant

To set up the Vagrant box:

	$ vagrant up

To subsequently run ansible over the box again:

	$ vagrant provision

Or, possibly quicker:

	$ ansible-playbook --private-key=.vagrant/machines/default/virtualbox/private_key --user=vagrant --connection=ssh --inventory-file=.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory vagrant.yml

When recreating the box I did get an error at this point, and doing this fixed it:

	$ ssh-keygen -R [127.0.0.1]:2222

Once it's run you can ssh in to Vagrant using the `deploy` user (using the IP address set in `Vagrantfile` and `inventories/vagrant.ini`):

	$ ssh deploy@192.168.33.15

or as the standard `vagrant` user:

	$ vagrant ssh


## DigitalOcean

1. Have an SSH key set on your account: https://www.digitalocean.com/community/tutorials/how-to-use-ssh-keys-with-digitalocean-droplets

2. Create a new Ubuntu 14.04 x64 droplet, clicking the checkbox to add your SSH key (or add a new one).

3. You should be able to do (in this and subsequent examples, change the IP address to your droplet's of course):

	```
	$ ssh root@188.166.146.145
	```

	If you get a warning about 'REMOTE HOST IDENTIFICATION HAS CHANGED!' after destroying and creating a new droplet, you can remove the warning with:

	```
	$ ssh-keygen -R 188.166.146.145
	```

4. Put the droplet's IP address in `inventories/production.ini`. eg:

	```
	[webservers]
	188.166.146.145
	```

5. Run the playbook (note, this first time we specify the user as `root`):

	```
	$ ansible-playbook --i inventories/production.ini -u root production.yml
	```

6. It should be all done. If the variable `ubuntu_use_firewall` is true (set in `env_vars/*.yml`), then you'll only be able to SSH to the `ubuntu_ssh_port` as the `ubuntu_deploy_user` eg, if the user is `deploy` and `ubuntu_ssh_port` is `1025`:

	```
	$ ssh deploy@188.166.146.145 -p 1025
	```

	These should fail (although the first will work if `ubuntu_ssh_port` is `22`, the default):

	```
	$ ssh deploy@188.166.146.145
	$ ssh root@188.166.146.145 -p 1025
	```

7. If the SSH port has now changed (as in the previous step), you'll need to add it to `inventories/production.ini`. eg:

	```
	[webservers]
	188.166.146.145:1025
	```

8. For subsequent runs, you'll need to set it to use the `ubuntu_deploy_user` and use `-s` to become sudo, and `-K` to be prompted for the sudo password (set in an `env_vars/*.yml` file):

	```
	$ ansible-playbook --i inventories/production.ini -u deploy -s -K production.yml
	```


## Notes

`roles/common/` is stuff to do with setting up the basic server, before we
get to webservers, databases, etc.

Each "app" (eg, a website) should have its variables set in the `apps` list in `roles/apps/vars/main.yml`. eg:

	apps:
	  - repo: git@github.com:philgyford/twelescreen.git
	    virtualenv: twelescreen
	  - repo: git@github.com:philgyford/myphpapp.git

If the app requires a python virtualenv, set its `virtualenv` name. Otherwise, leave that property out.

## TODO

* Set apps' environment variables?
* Create apps' databases
* Copy databse with scp?
* Restore database?
* If django: Transfer media files from local machine?
* If django: Collect static files.
* Nginx and gunicorn
* Install memcached-dev and memcached if needed
* Git ssh stuff for updates
* Set up postgres backups to s3?
* Vaulted secrets, using tip in https://www.reinteractive.net/posts/167-ansible-real-life-good-practices


## TODO LATER

* Make Vagrant more similar to live. Maybe http://hakunin.com/six-ansible-practices ?

* Improve firewall stuff for Vagrant. At the moment it's just 'off'. Would be good to have it more similar to live, but I got confused over configuring SSH.

* Rename pepysdiary `live.py` settings file?
