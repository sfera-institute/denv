# DENV

A nifty utility to manage development environments with Docker.

## Why?

I was working on several projects that required different application stacks, and it was annoyingly difficult
to manage. So, being a lazy and procrastinating programmer, I figured that's a problem worth solving, rather
than working on the projects themselves.

You know how Python has virtual environments? You just run ``virtualenv``, and suddenly your interpreter and
all your dependencies are nicely "encapsulated" in a designated directory? Well, I thought, why not do the
same for the entire system! Especially in this day and age, when we have [Docker](https://www.docker.com/),
which can easily emulate entire application stacks.

The only problem is that Docker was meant for *deploying* applications, not for *developing* them, so it's
a wee too cumbersome for daily life. This little tool, however (seriously, it's tiny: some 200 lines of bash)
solves just that, by introducing the ``denv`` command, and making the entire process easy as pie.

## Installation

Clone the repository, make sure the ``denv`` script is executable, and optionally drop it someplace on
your ``PATH`` (e.g. ``/usr/local/bin``) so you can call it from anywhere.

```sh
/home/dan$ git clone git@github.com:dan-gittik/denv.git
...
/home/dan$ chmod +x denv/denv
/home/dan$ sudo cp denv/denv /usr/local/bin

# And now denv should be available as a command...
/home/dan$ denv
USAGE: denv NAME [IMAGE] | --list | --save NAME | --stop NAME | --delete NAME
```

## Usage

To activate an environment, run ``denv NAME``.

```sh
/home/dan$ denv foo
[denv] Creating foo from ubuntu:latest...
[denv] Running foo...
[denv] Joining foo...
$ # Woot, I'm in the environment!
```

Now you can install whatever programs you like, write whatever files you like — and generally, do whataver you 
like — and everything will happen within that environment (i.e. its container).

```sh
$ apt update
...
$ apt install -y python3
...
$ echo "print('Hello, world.')" > app.py
$ python3 app.py
Hello, world.
```

Then, when you exit, everything is saved as part of that environment, so you can re-activate it and keep working 
on it later.

```sh
$ exit
[denv] Saving foo...
[denv] Stopping foo...
/home/dan$ python3
bash: python3: command not found # I don't even have Python 3 on the host!

# Later that day...
/home/dan$ denv foo
[denv] Running foo...
[denv] Joining foo...
$ python3 app.py
Hello, world. # Whoa, everything still works!
```

The files in the main directory (``/app``) are mapped to the current directory (i.e. wherever the ``denv``
command was invoked), so not only do they persist, but they can be edited outside the environment (e.g. in a 
graphical text editor on the host).

```sh
# Meanwhile, on the host...
/home/dan$ ls
app.py
/home/dan$ cat app.py
print('Hello, world.')
/home/dan$ vscode .
...
```

Furthermore, if the environment is already running (for example, on another terminal), the command will join 
that same environment, and stop it only when its last session exits.

```sh
# Terminal 1
/home/dan$ denv foo
[denv] Running foo...
[denv] Joining foo...
$ 

# Terminal 2
/home/dan$ denv foo
[denv] Joining foo...
$

# Terminal 1
$ exit
[denv] Saving foo...
/home/dan$

# Terminal 2: the session is still active!
$ exit
[denv] Saving foo...
[denv] Stopping foo... # Only now does it stop.
/home/dan$
```

To create an environment based on a specific image, simply specify it after the environment name:
``denv NAME IMAGE``.

```sh
/home/dan$ denv bar alpine:latest
[denv] Creating bar from alpine:latest...
latest: Pulling from library/alpine
...
```

To list all the existing environment, run ``denv --list``.

```sh
# Terminal 1
/home/dan$ denv --list
foo (inactive)
bar (inactive)

# Terminal 2: let's activate foo!
/home/dan/foo$ denv foo
[denv] Running foo...
[denv] Joining foo...
$

# Terminal 1: now foo is active, and we can even see where it's mapped.
/home/dan$ denv --list
foo (/home/dan/foo)
bar (inactive)
```

The environment is saved whenever a session exits, unless it exits with a non zero exit code (e.g. ``exit 1``).
To save an environment explicitly, run ``denv --save NAME``.

```sh
/home/dan$ denv --save foo
[denv] Saving foo...
```

The environment is stopped when its last session exits. To stop an environment explicitly (which will cause all 
its active sessions to exit), run ``denv --stop NAME``.

```sh
/home/dan$ denv --stop foo
[denv] Stopping foo...
```

To delete an environment, run ``denv --delete NAME``. Note that this may result in data loss, as everything in 
that environment will be deleted (expect for its main directory, which is persistently mapped to the host).

```sh
/home/dan$ denv --delete foo
[denv] Stopping foo...
[denv] Deleting foo...
```

To modify the environment configurations (for example, to publish port 80, so a webserver running on it can be 
accessed by a browser on the host), add the necessary Docker command line arguments to the ``/.denv.config`` file.

```sh
/home/dan$ denv foo
[denv] Running foo...
[denv] Joining foo...
$ echo '-p 8000:80' > /.denv.config

# We need to restart the environment for the changes to take effect...
$ exit
[denv] Saving foo...
[denv] Stopping foo...
/home/dan$ denv foo
[denv] Running foo...
[denv] Joining foo...
$ python3 -m http.server 80
...

# And now, try browsing http://localhost:8000/ on the host. Huzzah!
```

To modify the environment initialization (for example, start some services), add the commands to the 
``/.denv.init`` script.

```sh
/home/dan$ denv foo
[denv] Running foo...
[denv] Joining foo...
$ echo 'touch /app/example' > /.denv.init
$ ls
# Nothing; namely, no /app/example.

# We need to restart the environment for the initialization script to run...
$ exit
[denv] Saving foo...
[denv] Stopping foo...
/home/dan$ denv foo
[denv] Running foo...
[denv] Joining foo...
$ ls
example # Et voilá!
```

### Configurations

You can change `denv`'s configurations by exporting the right environment variables:

`DENV_REPO`
    The image's repository (defaults to `denv`).

`DENV_SHELL`
    The container's shell (default is `/bin/bash`).

`DENV_ROOT`
    The container's working directory (default is `/app`).

`DENV_DEFAULT_IMAGE`
    The default image used for new environments (default is `ubuntu:latest`).

`DENV_CONFIG_PATH`
    The path used for the container's configuration (default is `/.denv.config`).

`DENV_INIT_PATH`
    The path used for the container's initialization script (default is `/.denv.init`).

`DENV_WORKDIR_PATH`
    The path used for mapping the container to a host directory, used by `denv --list` (default is `/.denv.workdir`).

`DENV_LOG`
    The logging status: `0` for off, `1` for on (default is `1`).

`DENV_LOG_PREFIX`
    The logging prefix (default is `[denv] `).