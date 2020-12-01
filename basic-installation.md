# SATOSA 7.0.1 Basic Installation

In order to do anything useful with SATOSA you need at least to:
- Install the needed libraries and the main software components.
- Have a template configuration files set to play with.
- Create a set of certificates to use with the frontend(s) and backend(s).

Let's do it!

## The software

For SATOSA to work you will need the libraries libffi and libssl plus the headers. On Debian based 
distributions install both with apt-get:

```
apt-get install libffi-dev libssl-dev 
```

SATOSA can be installed with pip and it should be run in a python3 virtualenv. On a Debian based 
distro, install the following:

```
apt-get install python3 python3-venv python3-pip
```

Then create a directory for the SATOSA configuration files and the venv:

```
mkdir /opt/satosa
cd /opt/satosa
python3 -mvenv venv
```

Activate the venv and install SATOSA version 7.0.1 with pip:

```
cd /opt/satosa
. venv/bin/activate
pip install satosa==7.0.1
```

## Configuration files

Create an `etc` directory where to host the configuration files:

```
mkdir /opt/satosa/etc
```

Pull SATOSA 7.0.1 from the github repo so you can make use of the example configuration files:

```
cd /opt/satosa
mkdir releases
cd releases
wget https://github.com/IdentityPython/SATOSA/archive/v7.0.1.tar.gz
tar xfz v7.0.1.tar.gz
cp -r SATOSA-7.0.1/example/* /opt/satosa/etc
```

All the provided configuration files ends with `.example`, remember to remove it before using them,
or even better make a copy.

Independentely from which frontend and backend plugins combination you are going use, you will
certainly need a main configuration file, `proxy_conf.yaml`, and way to map attributes between
different components, which is defined in the `internal_attributes.yaml` file. So, let's create a
working copy of both files:

```
cd /opt/satosa/etc
cp proxy_conf.yaml.example proxy_conf.yaml
cp internal_attributes.yaml.example internal_attributes.yaml
```

## Keys and Certificates

For SATOSA to work you will need at least 4 pairs of certificate and key (after the colon you'll 
find the expected filenames):
- the HTTPS endpoints of the proxy: `https.crt` and `https.key`.
- the metadata signing: `metadata.crt` and `metadata.key`.
- the frontend plugins: `frontend.crt` and `frontend.key`.
- the backend plugins: `backend.crt` and `backend.key`.

The certificate and key pair for the HTTPS endpoints is better to be a certificate issued by a valid
CA, otherwise you'll have a hard time even in a test environment.

The other three pairs can be created in one run with the following command:

```
for ck in metadata frontend backend; do openssl req -new -newkey rsa:3072 -keyout /opt/satosa/etc/$ck.key -nodes -x509 -out /opt/satosa/etc/$ck.crt -subj /CN=PROXY_FQDN; done 
```
