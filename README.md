
DISCLAIMER - ABANDONED/UNMAINTAINED CODE / DO NOT USE
=======================================================
While this repository has been inactive for some time, this formal notice, issued on **December 10, 2024**, serves as the official declaration to clarify the situation. Consequently, this repository and all associated resources (including related projects, code, documentation, and distributed packages such as Docker images, PyPI packages, etc.) are now explicitly declared **unmaintained** and **abandoned**.

I would like to remind everyone that this project’s free license has always been based on the principle that the software is provided "AS-IS", without any warranty or expectation of liability or maintenance from the maintainer.
As such, it is used solely at the user's own risk, with no warranty or liability from the maintainer, including but not limited to any damages arising from its use.

Due to the enactment of the Cyber Resilience Act (EU Regulation 2024/2847), which significantly alters the regulatory framework, including penalties of up to €15M, combined with its demands for **unpaid** and **indefinite** liability, it has become untenable for me to continue maintaining all my Open Source Projects as a natural person.
The new regulations impose personal liability risks and create an unacceptable burden, regardless of my personal situation now or in the future, particularly when the work is done voluntarily and without compensation.

**No further technical support, updates (including security patches), or maintenance, of any kind, will be provided.**

These resources may remain online, but solely for public archiving, documentation, and educational purposes.

Users are strongly advised not to use these resources in any active or production-related projects, and to seek alternative solutions that comply with the new legal requirements (EU CRA).

**Using these resources outside of these contexts is strictly prohibited and is done at your own risk.**

This project has been transfered to Makina Corpus <freesoftware@makina-corpus.com> ( https://makina-corpus.com ). This project and its associated resources, including published resources related to this project (e.g., from PyPI, Docker Hub, GitHub, etc.), may be removed starting **March 15, 2025**, especially if the CRA’s risks remain disproportionate.

![Tests](https://github.com/obestwalter/tox-direct/workflows/Tests/badge.svg)

# tox-direct

[tox](https://tox.readthedocs.io) plugin: run tox envs directly in the same interpreter that tox was run in.

`tox-direct` is something that you usually shouldn't need, but that can be handy in certain situations. It is not recommended to use this approach as a normal part of a tox based automation workflow. The fact that tox creates isolated virtual environments for all automation activities is one of the main reasons for its reliability and is the key to unify command line based and CI automation. 

Having said that: life is messy and sometimes you just want to run a certain environment in the host interpreter. For this there is `tox-direct` now.

**TODO: maybe it makes sense to extend this to being able to run in the basepython rather than in the host environment (if they differ). This is not implemented as I have no need for it (yet).**

## installation

    pip install tox-direct

## concept

`tox-direct` is trying to be safe first and should also have the ability to degrade gracefully when `tox-direct` is not installed. To request that an environment is run directly it can be requested via command line argument, environment or setting `direct = True` in a testenv.

To be safe the following activities will be deactivated by default in direct runs:

* package build
* deps installations
* project installation

**WARNING: be aware though that the commands are run as is. If you install things in commands they will be installed in the host environment.**

There are two ways to request envs being run in direct mode: **static** and **on request**. The on request variant also provides a **YOLO** option ((you only live once ;)) which means that everything is run in the host interpreter. This will change the host interpreter and is usually only safe and makes sense (or works at all) if tox is run in a virtual environment already.

### static form 
if the testenv name contains the word **direct** it will be run in direct mode if tox-direct is installed. If this is part of a project that is shared you should make sure that this also works as intended if `tox-direct` is not installed (a.k.a. degrades gracefully).
if the testenv name contains the word **direct** or the attribute `direct = True` it will be run in direct mode if tox-direct is installed. If this is part of a project that is shared you should make sure that this also works as intended if `tox-direct` is not installed (a.k.a. degrades gracefully).

### on request form 

`tox-direct` adds command line arguments and checks environment variables to run arbitrary envs in direct mode.

With `tox-direct installed`:

```text
$ tox --help
[...]

optional arguments:
  --direct        [tox-direct] deactivate venv, packaging and install steps - 
                  run commands directly (can also be achieved by setting
                  TOX_DIRECT) (default: False)
  --direct-yolo   [tox-direct] do everything in host environment that would
                  otherwise happen in an isolated virtual environment 
                  (can also be achieved by setting TOX_DIRECT_YOLO env var
                  (default: False)
```

**WARNING: `tox-direct` does not consider different basepythons, which means that running environments with different basepythons makes no sense with `tox-direct` at the moment: they would all run in the same environment where tox is installed (effectively doing the same over and over again).**

## basic example

Let's assume you are working in some virtualenv already which looks like this:

```text
$ which python
~/.virtualenvs/tmp/bin/python

$ pip list
pip list
Package            Version
------------------ -------
[...] 
tox                3.13.1 
tox-direct         0.2.2  
[...]  


$ tox --version
3.13.1 imported from ~/.virtualenvs/tmp/lib/python3.6/site-packages/tox/__init__.py
registered plugins:
    tox-direct-0.2.2 at ~/.virtualenvs/tmp/lib/python3.6/site-packages/tox_direct/hookimpls.py
```

You have a project with a `tox.ini` like this:

```ini
[tox]
; this is the default - put here to be explicit
skipsdist = False

[testenv:direct-action]
; also the default to be explicit
skip_install = False
deps = pytest
commands =
    pip list
    which python

[testenv:env-attribute]
direct = True
; also the default to be explicit
skip_install = False
deps = pytest
commands =
    pip list
    which python

[testenv:normal]
whitelist_externals = which
skip_install = False
usedevelop = True
commands = which python
```

tun tox:

```text
$ tox -qr
Package            Version
------------------ -------
[...] 
tox                3.13.1 
tox-direct         0.2.2  
[...]  
/home/ob/.virtualenvs/tmp/bin/python
Package            Version
------------------ -------
[...]
tox                3.13.1
tox-direct         0.2.2
[...]
/home/ob/.virtualenvs/tmp/bin/python
Package            Version
------------------ -------
[...]
pytest             4.6.3
example-project    1.3
[...]
/home/ob/oss/tox-dev/tplay/.tox/normal/bin/python
_________________ summary _______________________
  direct-action: commands succeeded
  normal: commands succeeded
  congratulations :)
```

The `direct-action` env shows the packages from the `tmp` virtual env and pytest was not installed, the project itself was also not installed.

**WARNING: if something is installed in the commands (e.g. contains `pip install` calls) this will still happen as commands will be executed without further inspection.**
 
tox still creates an envdir at `.tox/direct-action` but it does not contain a virutal environment - it is only used for internal bookkeeping and logging. The interpreter used throughout is `~/.virtualenvs/tmp` - the host interpreter that tox was started from.

The `normal` env ran in the isolated environment provided by tox. pytest was installed and so was the project itself (because no package is needed to install the project in development mode). If usedevelop was set to `False` tox would crash with a note that you can't do this in direct mode (because sdist is not built in direct mode). 

run the normal environment in direct mode:

```text
tox -qre normal --direct
Package            Version Location                           
------------------ ------- --------
[...]  
tox                3.13.1  
tox-direct         0.2.2 
[...]   
/home/ob/.virtualenvs/tmp/bin/python
____________________ summary _______
  normal: commands succeeded
  congratulations :)
```

Thi time it ran in the host and nothing extra was installed.

And now the YOLO version:

```text
tox -qre normal --direct-yolo
Package            Version    Location                           
------------------ ---------- --------
[...]   
example-project    1.3   
pytest             4.6.3      
[...]     
tox                3.13.1     
tox-direct         0.2.2
[...]      
/home/ob/.virtualenvs/tmp/bin/python
______________________ summary _________________________________
  normal: commands succeeded
  congratulations :)
```

pytest and the project where installed in the host environment.

**NOTE: the YOLO option is called YOLO for a reason in case you run this as root in your system python :).**
