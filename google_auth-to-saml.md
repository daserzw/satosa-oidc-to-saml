# google_auth-to-saml

This notes describe how to quickly set up a SATOSA proxy for using Google OAuth 2.0/OpenID Connect
authentication to get access to a SAML Service Provider. 

This notes are based on SATOSA version 7.0.1 and have been tested on a Debian Buster distribution.

## Basic Installation

Please follow the basic installation document in order to set up SATOSA and the working environment:
[basic-installation.md](basic-installation.md)


## Architecture

The architecture that will be implemented is composed by:
* a SATOSA instance composed by:
  * Frontend: a SAML 2.0 Identity Provider.
  * Backend: an OIDC client for the Google Authentication.
* a SAML 2.0 Discovery Service (provided outside this notes).
* a SAML Service Provider belonging to the identity federation in which the IdP will be registered
  (provided outside this notes).


```

      +---------------+       +---------------+                                      
      |   SAML 2.0    |-------|  SAML 2.0     |                                      
      |      SP       |       |  Discovery    |                                      
      +---------------+       |  Service      |                                      
              |               +---------------+                                      
              |                                                                      
              |                                                                      
      +---------------+                                                              
      |SATOSA FRONTEND|                                                              
      | SAML 2.0 IdP  |                                                              
      +---------------+                                                              
      | SATOSA core & |                                                              
      | microservices |                                                              
      +---------------+                                                              
      |SATOSA BACKEND |                                                              
      |GOOGLE OIDC RP |                                                              
      +---------------+                                                              
              |                                                                      
              |                                                                      
              |                                                                      
      +---------------+                                                              
      | GOOGLE AUTHn  |                                                              
      | OIDC Provider |                                                              
      +---------------+                                                              

```

### Proxy configuration

The main confguration file is `proxy_conf.yaml`. The example configuration is almost ready for 
running the proxy, with some modifications:

- `BASE` should be set to the base URL of the proxy, as in:
 ```
 BASE: https://FQDN
 ```
 
- `BACKEND_MODULES` should point to the Google backend configuration file, as in:

 ```
 BACKEND_MODULES:
  - "plugins/backends/google_backend.yaml"
 ```

- `FRONTEND_MODULES` should point to the Saml 2 frontend configuration file, as in:

 ```
 FRONTEND_MODULES:
   - "plugins/frontends/saml2_frontend.yaml" 
 ```
- comment `MICRO_SERVICES` and its child element as we won't use any microservice in this example:
 ```
 #MICRO_SERVICES:
 #  - "plugins/microservices/static_attributes.yaml"
 ```
- modify the `cookies_samesite_compat` value as below to correct a typo in the example
 configuration:
 ```
 cookies_samesite_compat:
  - ["SATOSA_STATE", "SATOSA_STATE_LEGACY"]
 ```

The rest of the configuration file is good to go.

### Google backend 

The configuration of the Google is quite straightforward. Once you have logged in the google console
for developers, https://console.developers.google.com, select __Credentials__, then click on CREATE
CREDENTIALS, select OAuth client ID and __Web Application__ as application type. The parameters that
you need to register a client on the Google console for developers are only the __Authorised
JavaScript origins__ which correspond to your HTTP origin, that is `https://FQDN[:port]` and the
__Authorised redirect URIs__ which by default is `https://FQDN[:port]/google`. 

Once you have created the OAuth client you will get the Client ID and the Client Secret that you
should use in the `plugins/backends/google_backend.yaml` file as follows:

- set the `client_id` and the `client_secret` that you have got from the google n the corresponding key:
 
 ```
    client_metadata:
      client_id: SOMETHING-SOMETHING.apps.googleusercontent.com 
      client_secret: ALPHA-NUMERIC-STRING

 ```
 
### SAML Frontend

The SAML Frontend is a SAML 2.0 Identity Provider (IdP) and it is configured through the file
`plugins/frontends/saml2_frontend.yaml`. Once the IdP will be set up we will create the
corresponding metadata. The configuration of the SAML 2.0 IdP is quite detailed and it has many
different options that deal with the source of trusted Service Providers metadata, the IdP user
interface information, the SAML 2.0 protocol and the registered endpoints. Let's start:

- the first things you want to set are the organisation details and the contacts, for example for
  the ACME Corporation:

```
  idp_config:
    organization: {display_name: The fabolous ACME Corporation, name: ACME Corporation, url: 'https://acme.corp'}
    contact_person:
    - {contact_type: technical, email_address: technical@acme.corp, given_name: Technical}
    - {contact_type: support, email_address: support@acme.corp, given_name: Support}
```

- the next information are about the metadata of the Service Providers (SPs) with which the IdP will talk
  to. There are some options. In the case of a single or a few SPs you will probably exchange the
  metadata file offline ant then you will configure them locally as in:
  
  ```
  metadata:
	local [sp.xml]
  ```  
  
  Or you would like to use the metadata distribution service of an identity federation and the
  corresponding public cert to verify the metadata signature (in that case your IdP is usually
  registered in that federation, but that's another story), as in:
  
  ```
  metadata:
    remote: {url:metadata-url, cert: path-to-the-cert.pem}
  ```
- then there is the time skew relative to the SP that the IdP will tolerate. The default is 60
seconds, which might be low in some contexts, so it would be better to raise it to 5 minutes:

 ```
 accepted_time_diff = 300
 ```
 
- then there is the service section where you want to set the IdP name and the UI information and
the logo as they may be displayed to the final user. So you want to customize `idp.name`, and
basically all the `ui_info` fields:

```
      idp:
        endpoints:
          single_sign_on_service: []
        name: Proxy IdP
        ui_info:
          display_name:
            - lang: en
              text: "IdP Display Name"
          description:
            - lang: en
              text: "IdP Description"
          information_url:
            - lang: en
              text: "http://idp.information.url/"
          privacy_statement_url:
            - lang: en
              text: "http://idp.privacy.url/"
          keywords:
            - lang: se
              text: ["Satosa", "IdP-SE"]
            - lang: en
              text: ["Satosa", "IdP-EN"]
          logo:
            text: "http://idp.logo.url/"
            width: "100"
            height: "100"
```

- the `name_id_format` default is good enough for most cases and if you know what a SAML 2.0
  persistent id is and why you don't want it, go set it to just the `transient` one.
  
- the `policy` section contains some fundemental technical features, such as the duration of the
  SAML Response, `lifetime`, and wheter to encrypt or not the assertions, `encrypt_assertion`. This
  is realy related to the context in which the IdP will be used, the relative policies (such as a
  federation policy), the intended use and what the SP is expecting. That said, the default is not
  to encrypt the assertion which is ok in a test environment, but you might want to change it when
  you into production.
  
- the rest is good to go.

### Create and fix the SAML metadata

Once all the SAML 2.0 configuration files are set up, it's time to create the metadata with the 
`satosa-saml-metadata` command:

```
cd /opt/satosa/etc
satosa-saml-metadata proxy_conf.yaml metadata.key metadata.crt
```

The command will write the SAML 2.0 Identity Provider metadata, the proxy frontend, into the file 
`frontend.xml`. Depending on how you want to use the IdP, it might be enough to exchange the
metadata file with the SP or you might want to go all the way down and register the IdP in an
identity federation. In both cases, it might be that some defaults of the metadata file created by
SATOSA will get in the way, so let's see how to get rid of them.

For your convenience format the file with `xmllint` for better readability and editing:

`xmllint -format frontend.xml > frontend-edited.xml`

Open the last file with your preferred editor and:
- delete all the `<ns1:Signature Id="Signature1">` element. You do not need any signature because 
  registration in identity federation is happening out of band and your metadata will be signed 
  with the federation key once it has been validated.

- delete the `use="signing"` attribute in the `<ns0:KeyDescriptor` tag. 

Now you are ready to exchange the metadata.

### Run the proxy

It's finally time to run the proxy, but before that you want to check that:
- the Google client credentials are active.
- the SAML 2.0 Identity Provider metadata have been exchanged with the Service Provider(s) with
  which you want to talk to.

To run the proxy as WSGI application with `gunicorn` use the following command:
```
cd /opt/satosa/etc
. ../venv/bin/activate
gunicorn -b0.0.0.0:443 satosa.wsgi:app --keyfile=https.key --certfile=https.crt > gunicorn.log 2>&1 &
```

Both the `stdout` and `stderr` are redirected into the file `gunicorn.log`. 

Have fun and please open an issue or drop me a line at <davide.vaghetti@garr.it> if you spot any error.
