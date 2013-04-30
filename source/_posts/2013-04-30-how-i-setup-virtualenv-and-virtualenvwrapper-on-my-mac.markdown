---
layout: post
title: "How I Setup VirtualEnv and VirtualEnvWrapper on my Mac"
date: 2013-04-30 14:23
comments: true
categories: python mac virtualenv virtualenvwrapper
---

The following are the steps I take to setup up [VirtualEnv](http://www.virtualenv.org/en/latest/) and [VirtualEnvWrapper](http://virtualenvwrapper.readthedocs.org/en/latest/) on a new machine.

* Make sure pip is installed by running pip in the terminal.

* If it isn't installed, install by doing

```    
sudo easy_install pip
```

* Next, install virtualenv and virtualenvwrapper

```
sudo pip install virtualenvwrapper
```

```
sudo pip install virtualenvwrapper
```

* Open up your .bash_profile or .profile, and after your PATH statement, add the following

    {% codeblock lang:bash %}
    # set where virutal environments will live
    export WORKON_HOME=$HOME/.virtualenvs
    # ensure all new environments are isolated from the site-packages directory
    export VIRTUALENVWRAPPER_VIRTUALENV_ARGS='--no-site-packages'
    # use the same directory for virtualenvs as virtualenvwrapper
    export PIP_VIRTUALENV_BASE=$WORKON_HOME
    # makes pip detect an active virtualenv and install to it
    export PIP_RESPECT_VIRTUALENV=true
    if [[ -r /usr/local/bin/virtualenvwrapper.sh ]]; then
        source /usr/local/bin/virtualenvwrapper.sh
    else
        echo "WARNING: Can't find virtualenvwrapper.sh"
    fi
    {% endcodeblock %}

* Open a new terminal window.  You should see virtualenvwrapper.sh being run and setting up your .virtualenvs directory.

* Test creating a new virtualenv

```
$ mkvirtualenv testenv
```

* You should see something in the console like

```
New python executable in testenv/bin/python
Installing setuptools............done.
Installing pip...............done.
virtualenvwrapper.user_scripts creating /Users/kelsmj/.virtualenvsexport/testenv/bin/predeactivate
virtualenvwrapper.user_scripts creating /Users/kelsmj/.virtualenvsexport/testenv/bin/postdeactivate
virtualenvwrapper.user_scripts creating /Users/kelsmj/.virtualenvsexport/testenv/bin/preactivate
virtualenvwrapper.user_scripts creating /Users/kelsmj/.virtualenvsexport/testenv/bin/postactivate
virtualenvwrapper.user_scripts creating /Users/kelsmj/.virtualenvsexport/testenv/bin/get_env_details
```

* That should put you into a new virtualenv called "testenv"

* Lets install some packages and create a requirements.txt file

```
$ pip install rq
$ pip install requests
$ pip install google-api-python-client
$ pip freeze > requirements.txt
```

* Your have just installed 3 python packages into your virtual environment, and created a .txt file that will have all the necessary information in order to reproduce your environment.

* You can leave your virtualenv by doing

```
deactivate testenv
```

* To create a new virtualenv, and install the libraries based on the requirements.txt file you just created, just do the following.

```
mkvirtualenv testenv2
pip install -r requirements.txt
```

* List of commonly used commands
   * mkvirtualenv (create a new virtualenv)
   * rmvirtualenv (remove an existing virtualenv)
   * workon (change the current virtualenv)
   * add2virtualenv (add external packages in a .pth file to current virtualenv)
   * cdsitepackages (cd into the site-packages directory of current virtualenv)
   * cdvirtualenv (cd into the root of the current virtualenv)
   * deactivate (deactivate virtualenv, which calls several hooks)
