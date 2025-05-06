+++
date = "2025-03-12T19:33:43Z"
draft = true
title = "Hello Django"
tags = ["python"]
+++

## Prerequisites
Being a `Macbook` user means dependencies are for `brew` but this blog post should work with any `unix` distribution.
- [homebrew](https://brew.sh/)
- [pip](https://pip.pypa.io/en/stable/installation/)

## python environment setup
```
❯ pip install pipenv
Defaulting to user installation because normal site-packages is not writeable
Collecting pipenv
  Using cached pipenv-2024.4.1-py3-none-any.whl.metadata (17 kB)
Using cached pipenv-2024.4.1-py3-none-any.whl (3.0 MB)
Installing collected packages: pipenv
Successfully installed pipenv-2024.4.1
```

## django installation
```
❯ pipenv install django shell
Creating a virtualenv for this project
✔ Successfully created virtual environment!
Virtualenv location: ~/.local/share/virtualenvs/ido-xOnlinehW

Creating a Pipfile for this project...
Pipfile.lock not found, creating...
Locking [packages] dependencies...
Locking [dev-packages] dependencies...

Installing django...
✔ Installation Succeeded
Installing shell...
✔ Installation Succeeded
```

## django project
```
❯ django-admin startproject bigday .
❯ django-admin startapp guests
❯ lt
 .
├──  bigday
├──  guests
├──  manage.py
├──  Pipfile
└──  Pipfile.lock
```

### Running migration
```
❯ pipenv shell
Launching subshell in virtual environment...

❯ python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying sessions.0001_initial... OK
```

### Creating superuser
```
❯ python3 manage.py createsuperuser
Username (leave blank to use 'whoami'): lekat
Email address: admin@lekat.uk
Password:
Password (again):
Superuser created successfully.
```

### Running WSGI server
```
❯ python3 manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
March 12, 2025 - 21:55:45
Django version 4.2.20, using settings 'bigday.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```