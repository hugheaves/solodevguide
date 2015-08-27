# Installing Files and Code

## Uploading Files

`rsync` is the preferred tool for synchronizing code and files between your desktop and Solo. e.g.

```sh
rsync -avz local/file/path/. root@10.1.1.10:/solo/path/. 
```

## Installing Packages

Solo is an `rpm` based system. These packages can be managed by the `smart` package system, already installed on your Solo.

[After having installed the `sdg` tool](utils.html), from your Solo's shell run:

```sh
sdg tunnel-start
```

This tunnels an Internet connection to Solo through your computer. Now we can configure the `smart` package manager to download packages from a dedicated package repository.

Run this command to add the package repository:

```sh
smart channel --add solo type=rpm-md baseurl=http://solo-packages.s3-website-us-east-1.amazonaws.com/3.10.17-rt12/
```

Now download the package list:

```sh
smart update
```

You can now use `smart search` and `smart install <package>` to install packages. You will see examples used throughout this guide.

These packages are precompiled and provided by 3DR for your use. To compile other packages may require rebuilding the Yocto Linux distribution.

**NOTE:** If you want to restore your package manager state after it's been modified, you can reset it by brute force:

```
smart channel --show --remove-all
yes | smart channel --add mydb type=rpm-sys name="RPM Database" 
yes | smart channel --add solo type=rpm-md baseurl=http://solo-packages.s3-website-us-east-1.amazonaws.com/3.10.17-rt12/
```

## Working with Python

Python 2.7 is used throughout our system and in many of our examples. There are a few ways in which you can deploy Python code to Solo.

Be aware: some Python libraries are "binary" dependencies. Solo does not ship a compiler (by design) and so cannot install code that requires C extensions. Trial and error is adequate to discovering which packages are usable.

TODO: Provide ways of compiling code directly for Solo in the "Advanced Topics" section.

### Installing using Solo's package manager

Some Python libraries are provided by Solo's internal package manager. For example, `opencv` can be installed via `smart`, which provides its own Python library. After running:

```
smart install opencv
```

You can see its Python library is installed:

```
python
>>> import cv
>>> cv.__name__
'cv'
```

### Bundling Python code

This mechanism bundles Python code locally on your computer and expands it in a virtual environment on Solo. This has two benefits:

1. No Internet connection or reliance on package management on Solo is needed.
2. Packages are installed in a virtual environment, so they don't collide with the global Solo namespace.

Start in a new directory on your host computer. This will be the entire directoy we send to Solo, so create your Python scripts here. We'll start by creating a virtual environment for local use:

```
pip install virtualenv
virtualenv env
source ./env/bin/activate
```

Now we can install Python packages using `pip install`. When you're ready to move code over to Solo, you want to create a `requirements.txt` file containing what packages you've installed:

```
pip freeze > requirements.txt
```

Next, we want to download these packages on your host computer so they can be moved to Solo along with your code. You can do this using `pip wheel` to download them into a new folder, `./wheelhouse`. Run this command:

```
CC=dummy pip wheel pytest -r ./requirements.txt --build-option="--plat-name=py27"
```

This installs all the dependencies in `requirements.txt` as Python wheel files, which are source code packages. The environment variable and platform options ensure we won't compile any host-only code, only source code.

Next, you can move this entire directory over to Solo using rsync:

```
rsync -avz --exclude="*.pyc" --exclude="env" ./my_python_code solo:/opt/my_python_code
```

SSH into Solo and navigate to the newly made directory (above `/opt/my_python_code`). Finally, run these commands:

```
virtualenv env
source ./env/bin/activate
pip install --no-index --find-links=wheelhouse/ -r requirements.txt
```

This requires no Internet connection. Instead, it installs from all the downloaded dependencies you transferred from your computer. You can now run your Python scripts with any pacakges you depended on, without having impacted any of Solo's own Python dependencies.


### Installing packages directly with pip

You can install code directly from pip on Solo. Note that this is useful for development, but not recommended for distributing code.

Having installed the `sdg` utility, run:

```
sdg install-pip
```

This will install and update pip to the latest version. You can then install any packages you like:

```
pip install virtualenv
```

This will install the package globally, so you may consider using `virtualenv` to create isolated environments of packages.