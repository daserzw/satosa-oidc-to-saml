# satosa-oidc-to-saml

This _not-that-brief_ notes are the outcome of an effort to understand how to quickly set up a 
__Proof of Concept__ SATOSA proxy as a simple OIDC to SAML 2.0 gateway for Research and Education 
Identity Federation. Please consider that the author is __not an expert__ user of SATOSA and what 
follows __MUST NOT__ be used in a production environment without modifications.

References:
* https://github.com/IdentityPython/SATOSA
* https://lists.sunet.se/pipermail/satosa-users/
* https://lists.sunet.se/pipermail/satosa-dev/

This notes are based on SATOSA version 6.1.0.

## Architecture

The architecture that will be implemented is composed by:
* an OpenID Connect Relying Party (provided outside this notes).
* a SATOSA installation with:
  * an OpenID Connect Frontend.
  * a SAML 2.0 Service Provider.
* a SAML 2.0 Discovery Service (provided outside this notes).
* any SAML 2.0 Identity Provider belonging to the identity federation in which the SP is registered. 

```
      +---------------+
      |OpenID Connect |
      |Relying Party  |
      +---------------+
              |
              |
              |
      +---------------+
      |SATOSA FRONTEND|
      | OIDC Provider |
      +---------------+
      | SATOSA core & | 
      | microservices | 
      +---------------+       +--------------+
      |SATOSA BACKEND |       |  SAML 2.0    |
      | SAML 2.0 SP   |------ |  Discovery   |
      +---------------+       |  Service     |
              |               +--------------+
              |
              |
      +---------------+
      |   SAML 2.0    |
      |   Identity    |
      |   Provider    |
      +---------------+
```

## Installation

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

Activate the venv and install SATOSA version 6.1.0 with pip:

```
cd /opt/satosa
. venv/bin/activate
pip install satosa==6.1.0
```

## Configuration

Create an `etc` directory where to host the configuration files:

```
mkdir /opt/satosa/etc
```

Pull SATOSA 6.1.0 from the github repo so you can make use of the example configuration files:

```
cd /opt/satosa
mkdir releases
cd releases
wget https://github.com/IdentityPython/SATOSA/archive/v6.1.0.tar.gz
tar xfz v6.1.0.tar.gz
cp -r SATOSA-6.1.0/example/* /opt/satosa/etc
```

All the provided configuration files ends with `.example`. Let's make a copy of the basic set of 
files that will be used:

```
cd /opt/satosa/etc
cp proxy_conf.yaml.example proxy_conf.yaml
cp internal_attributes.yaml.example internal_attributes.yaml
cp plugins/frontends/openid_connect_frontend.yaml.example plugins/frontends/openid_connect_frontend.yaml
cp plugins/backends/saml2_backend.yaml.example plugins/backends/saml2_backend.yaml
```

### Keys and Certificates

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
for ck in metadata frontend backend; do openssl req -newkey rsa:3072 -keyout /opt/satosa/etc/$ck.key -nodes -out /opt/satosa/etc/$ck.crt -subj /CN=PROXY_FQDN; done 
```

### Proxy configuration

The main confguration file is `proxy_conf.yaml`. The example configuration is almost ready for 
running the proxy, with some modifications:

- `BASE` should be set to the base URL of the proxy, as in:
 ```
 BASE: https://FQDN
 ```
- `FRONTEND_MODULES` should point to the openid frontend configuration file, as in:
 ```
 FRONTEND_MODULES:
   - "plugins/frontends/openid_connect_frontend.yaml" 
 ```
- comment `MICRO_SERVICES` and its child element as we won't use any microservice in this example:
 ```
 #MICRO_SERVICES:
 #  - "plugins/microservices/static_attributes.yaml"
 ```
- modify the `cookies_samesite_compat` value as below to correct a typo in the example configuration:
 ```
 cookies_samesite_compat:
  - ["SATOSA_STATE", "SATOSA_STATE_LEGACY"]
 ```

The rest of the configuration file is good to go.

### SAML 2.0 backend 

The next configuration file to set up is the `saml2_backend.yaml`, that configures the SAML 2.0
Service Provider side of the proxy as well as being used to create the SP metadata needed to 
register the SP in your SAML 2.0 identity federation (it can also be exchanged with just one IdP, 
but that is out of the scope of this tutorial). 

All the values below are good for registering the Service Provider in the IDEM Test Federation, 
YMMV. To have a minimal working configuration:

- comment the `idp_blacklist_file` item as in:
 
 ```
  #idp_blacklist_file: /path/to/blacklist.json
 ```
 
- insert the URL from where to download the aggregated metadata file of the identity federation 
  you wish to join and the certificate filename to verify the metadata signature, the path is 
  relative to `/opt/satosa/etc` (do not forget to download the certificate):
 
 ```
     metadata:
      remote: 
	    - {url: http://md.idem.garr.it/metadata/idem-test-metadata-sha256.xml, cert: idem-signer-20220121.pem}
 ```
 
- insert all the user interface information that will go into the metadata:

 ```
     service:
      sp:
        ui_info:
          display_name:
            - lang: en
              text: "IDEM SP Proxy TEST"
          description:
            - lang: en
              text: "IDEM SP Proxy TEST"
          information_url:
            - lang: en
              text: "https://wiki.idem.garr.it/wiki/About_(EN)"
          privacy_statement_url:
            - lang: en
              text: "https://wiki.idem.garr.it/wiki/Privacy_Policy_(EN)"
          keywords:
            - lang: it
              text: ["IDEM", "SP-Proxy"]
            - lang: en
              text: ["IDEM", "SP-Proxy"]
          logo:
            text: "https://www.idem.garr.it/images/logoIDEM_colore.svg"
            width: "130"
            height: "88"

 ```

- set `authn_requests_signed` to `false`, which will cause the SP _NOT_ to sign the AuthnRequest 
sent to the Identity Provider which is the default in the R&E Identity Federation:
 ```
     service:
	 [..]
       authn_requests_signed: false
 ```

- insert the URL of your discovery service in `disco_srv`:
 ```
   disco_srv: https://wayf.idem-test.garr.it/WAYF 
 ```

### Create and fix the SAML metadata

Once all the SAML 2.0 configuration files are set up, it's time to create the metadata with the 
`satosa-saml-metadata` command:

```
cd /opt/satosa/etc
satosa-saml-metadata proxy_conf.yaml metadata.key metadata.crt
```

The command will write the SAML 2.0 Service Provider metadata, the proxy backend, into the file 
`backend.xml`. The file needs to be edited before it can be used to register the SP in your identity
federation as SATOSA has some defaults that are quite well accepted by IdPs and aggregator metadata.

For your convenience format the file with `xmllint`, so the editing will be smoother:

`xmllint -format backend.xml > backend-edited.xml`

Open the last file with your preferred editor and:
- delete all the `<ns1:Signature Id="Signature1">` element. You do not need any signature because 
  registration in identity federation is happening out of band and your metadata will be signed 
  with the federation key once it has been validated.

- delete the `use="signing"` attribute in the `<ns0:KeyDescriptor` tag. 

Once you've done the above editing, you are ready to upload the metadata to your identity federation
registry.

### OpenID Connect frontend

The OpenID Connect frontend configuration file is named 
`plugins/frontend/openid_connect_frontend.yaml` and it will set up the OpenID Connect Provider side 
of the proxy that will be used to authenticate your OpenID Connect Relying Party. For a minimal 
working configuration you need to:

- comment `db_uri`:
 ```
 # db_uri: mongodb://db.example.com
 ```

- set the `client_db_path` to a JSON file where you will insert the details of your client (see 
below):
 
 ```
 client_db_path: cdb.json
 ```

- comment all the `extra_scopes` section:
 ```
 #    extra_scopes:
 #      foo_scope:
 #        - bar_claim
 #        - baz_claim
 ```

You will also need to create the file `cdb.json`:

```
cat << EOF > cdb.json
{
  "YOUR_CLIENT_ID": {
    "response_types": ["code"],
    "client_secret": "YOUR_CLIENT_SECRET",
    "redirect_uris": [
      "A THE CLIENT REDIRECT URI, USUALLY IS SOMETHING LIKE https://CLIENT_FQDN/auth/callback"
    ]
  }
}
EOF

```

Obviously `cdb.json` needs to be filled up with the correct values for your client(s). Most of the 
OIDC clients support the `code` response type, so you'll just need to set the client\_id, which is 
the _key_ of the client entry (`"YOUR_CLIENT_ID"`), the `client_secret` and the `redirect_uris`.

### Run the proxy

It's finally time to run the proxy, but before check that:
- the SAML 2.0 Service Provider side of the proxy, the backend, is registered in your identity federation.
- the OpenID Connect Relying Party you wish to authenticate is correctly set up with the `client_id` and the `client_secret` you have 
  written into `cdb.json`.

To run the proxy as WSGI application with `gunicorn` use the following command:
```
cd /opt/satosa/etc
. venv/bin/activate
gunicorn -b0.0.0.0:443 satosa.wsgi:app --keyfile=https.key --certfile=https.crt > gunicorn.log 2>&1 &
```

Both the `stdout` and `stderr` are redirected into the file `gunicorn.log`. 

Have fun and please open an issue or drop me a line at <davide.vaghetti@garr.it> if you spot any error.
