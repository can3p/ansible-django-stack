
ansible-django-stack
====================

Ansible Playbook designed for environments running a Django app.
It can install and configure these applications that are commonly used in
production Django deployments:

- Nginx
- Gunicorn
- PostgreSQL
- Supervisor
- Virtualenv
- Memcached
- Celery
- RabbitMQ
- Redis

*Note*: Compared to the [parent](https://github.com/jcalazan/ansible-django-stack), this fork makes some assumptions about
the setup and hense some variables are removed. Some parts of readme are also
remove, but it's not to hide the relation, but to reflect changes.

For example:

- No password for sudo
- Only letsencrypt for https
- Only https exposed
- We don't have defaults, every environment is ment to be a separate project with
  separate servers, so it should contain all the vars in it, no inheritance
- Keys are specified per environment, no default
- root user if forbidden
- anything project specific is removed/changed
- site host is separate from inventory hostname
- superuser is automatically created
- Runs django cookiecutter (didn't check with a clean setup)
- Additional step to run npm build to build static assets
- Basic auth for the service if needed

Breaking changes expected, use on your own risk.

## Variables

Check [example.yml](env_vars/example.yml) and [per_project/example](per_project/example) files
for configuration and hosts respectively. Please review the files and copy them for your project

Current setup only works with ssh agent forwarding enabled, so please keep this setting
for local deploys. In future, there will be a separate flow to deploy everything from
ci/locally.

```
ssh_forward_agent: true
```

## First things first

First of all, you'll need to setup a separate inventory file for your project. Something like
`per_project/example` with the following contents:

```
[all:vars]
env=example

[webservers]
example.com nginx_use_letsencrypt=true

[dbservers]
example.com
```

Contrary to parent this fork assumes that security playbook is run before anything else.

```
ansible-playbook -i per_project/example security.yml
```

After this you need to add db servers and after that deploy actual web servers. Please note that
although host groups are different in inventory file, current setup makes an assumption that
everything is on one box.

```
ansible-playbook -i per_project/example dbservers.yml
```

```
ansible-playbook -i per_project/example webservers.yml
```

Superuser is created automatically with `django_admin_user`, `django_admin_email`
and `django_admin_password` credentials.

You see example everywhere and it's not a coincidence.

To do deploys run `webservers` playbook with deploy tag:


```
ansible-playbook -i per_project/example webservers.yml -t deploy
```

### Configuring your application

It's expected that you will generate your project with
[Django cookiecutter](https://cookiecutter-django.readthedocs.io/en/latest/).

You can safely remove any production stuff from it, except config.settings.production
module.


## Security role tasks**

The security module performs several basic server hardening tasks. Inspired by
[this blog post][securing-ubuntu]:

- Updates `apt`
- Performs `aptitude safe-upgrade`
- Adds a user specified by the `server_user` variable, found in `roles/base/defaults/main.yml`
- Adds authorized key for the new user
- Installs sudo and adds the new user to sudoers with the password specified by
  the `server_user_password` variable found in `roles/security/defaults/main.yml`
- Installs and configures various security packages:
     - [Unattended upgrades](https://help.ubuntu.com/lts/serverguide/automatic-updates.html)
     - [Uncomplicated Firewall](https://wiki.ubuntu.com/UncomplicatedFirewall)
     - [Fail2ban](http://www.fail2ban.org/) (NOTE: Fail2ban is disabled by default
       as it is the most likely to lock you out of your server. Handle with care!)
- Restricts connection to the server to SSH and http(s) ports
- Limits `su` access to the sudo group
- Disallows password authentication (be careful!)
- Disallows root SSH access (you will only SSH to your machine as your new user
  and use a password for `sudo` access)
- Restricts SSH access to the new user specified by the `server_user` variable
- Deletes the `root` password


### Separate notes

Psql access on the db box:

```
$ PGPASSWORD=password psql -h localhost -U dbname
```

## TODO:

* Email settings
* Backups
* Removing RabbitMQ in favor of redis
* Testing with a clean cookiecutter project


[securing-ubuntu]: https://www.codelitt.com/blog/my-first-10-minutes-on-a-server-primer-for-securing-ubuntu/

## Credits

Nothing would be possible without fantastic parent project: https://github.com/jcalazan/ansible-django-stack
