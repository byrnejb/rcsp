# rcsp

A revised perl script to manage a small private Certificate Authority (CA)
based on CSP-0.3.4 by *Leif Johansson*.
Revised by James B. Byrne <byrnejb@harte-lyne.ca>

This is a command line driven application used to create and administer
a private Public Key Infrastructure (PKI).

This script is simply a small wrapper around OpenSSL.
It likely will not scale well but for small organisations requiring
a publiclly accessable Certificate Policy (CP) and
Certificate Practices Statement (CPS) together with providing
web accessible and downloadable PEM and DER formats for fewer
than several hundred generated certificates this should suffice.

More complete documentation can be obtained from cspguide.pdf found
in the `docs` directory.

# Getting started

Administrating Public Key Infrastructure (PKI) Certificate Authorities (CA)
using CSP.

2016-Nov-01 byrnejb@Harte-Lyne.ca

This procedure creates an `X509` PKI with a offline root CA and separate
issuing CAs. All CA's should be kept offline when not actively in use.
Whenever a CA is in use then the host employed should be first disconnected
from all networks (LAN and Internet). It will likely prove most convenient
to keep the CA directory trees on removable media such as USB flash
memory sticks or SD cards. These must be stored in a secure location and
only inserted into the host used to manage the PKI when required.

It is advised that a separate user id be created solely for the purpose
of administering the PKI.  Failing that then the root user should be used.
In either case care must be taken that any situation where the private
keys might be compromised is never permitted to arise.  That implies that
the medium holding the CA keys is never left unsecured; or unobserved when
in use; or inserted into a host that has multiple user access enabled.

# Creating a root CA

Some browsers no longer allow self-signed certificates to be added their
trusted certificate store.  Thus it is necessary that a private CA use a
different CA than their root CA to issue end use certificates.  Generally
speaking, the PKI root CA is only required to create additional issuing CA's
or replacements for compromised issuing CA's.

## Add the csp script location to your PATH

```
PATH=$PATH:/path/to/csp/
```

## Create the CA home directory.

```
mkdir -p /path/to/my-ca
cd /path/to/my-ca
```

## Copy the csp/ca/etc directory tree from the csp install location.

```
cp -pr /path/to/csp/ca/etc ./etc
```

## Create the root CA.

```
csp CA_HLL_ROOT_2016 create
```

Copy a localised extensions.conf template file into ./csp or modify
the default extensions.conf in place. At a minimum you need to change
all *example.com* references to your local domain.

```
cp ./extensions.conf.ca_hll_root ./csp/CA_HLL_ROOT_2016/extensions.conf
```

## Initialise the root CA.

This can be done in one step if one does not care what time of day the
certificate expires. However, if that route is taken then there is no
entry made in the root's certificate database for its own certificate
and the time of expiry is the same as the time of day when its
own certificate is created.

To specify an exact time of day to expire (i.e. midnight UTC) one
cannot use the default `openssl ca` command as that only takes as an
argument the number of days to expiry. Instead issue a certificate
signing request by specifying a csr file. Then self sign that request
providing the exact expiry date, time and timezone.  This requires
that we first capture the csp generated `openssl.conf` file since
`openssl` requires a vaild config file to operate.

The temporary config file generated by csp may be retained by defining
the environment variable `CSPDEBUG`. The resulting `csp-9999.conf` file
is found in `./csp/CA_HLL_ROOT_2016/tmp/`. The exact serial number
will vary and a new `csp-xxxx.conf` file created each time a csp command
is issued.

```
export CSPDEBUG=true # any value or none, it just needs to be defined.
```

## Initialise the root CA

Create a signing request file `ca.csr` instead of automatically self-signing
the CA certificate by specifying `--csrfile=`.

```
csp CA_HLL_ROOT_2016 init \
  --type=root \
  --keysize=4096  \
  --digest=sha512 \
  --url=ca.harte-lyne.ca \
  --email=certificates@harte-lyne.ca \
  --csrfile=./csp/CA_HLL_ROOT_2016/csrs/ca.csr \
  "CN=CA_HLL_ROOT_2016,OU=Networked Data Services,O=Harte & Lyne Limited,L=Hamilton,ST=Ontario,C=CA,DC=harte-lyne,DC=ca"
```

Copy the generated conf file from `./csp/CA_HLL_ROOT_2016/tmp`
to `./CA_HLL_ROOT_2016.conf`

```
ls -l ./csp/CA_HLL_ROOT_2016/tmp/*conf
-rw-rw-r--. 1 cspuser cspuser 2966 Nov  1 13:52 ./csp/CA_HLL_ROOT_2016/tmp/csp-9999.conf

cp ./csp/CA_HLL_ROOT_2016/tmp/csp-9999.conf ./CA_HLL_ROOT_2016.conf
```

## Self-sign the request using the copied configuration file.

Note that the start and end date arguments have the expected effect
on the certificate validity date and time.

```
openssl ca  \
  -selfsign  \
  -config ./CA_HLL_ROOT_2016.conf \
  -startdate 20161101000000Z  \
  -enddate 20361031235959Z \
  -keyfile ./csp/CA_HLL_ROOT_2016/private/ca.key \
  -infiles ./csp/CA_HLL_ROOT_2016/csrs/ca.csr
```

This creates the first certificate, 01.pem, in the
`./csp/CA_HLL_ROOT_2016/certs/` directory.
Copy `./csp/CA_HLL_ROOT_2016/certs/01.pem to /csp/CA_HLL_ROOT_2016/ca.crt`.
Do not delete, remove or rename `./csp/CA_HLL_ROOT_2016/certs/01.pem`.
Manually manipulating files in the CA database has a very real risk
of damaging the PKI to the point of uselessness.

```
cp ./csp/CA_HLL_ROOT_2016/certs/01.pem to /csp/CA_HLL_ROOT_2016/ca.crt
```

## Create the issuing CA key and certificate.

```
csp CA_HLL_ROOT_2016 request \
  --type=ca  \
  --keyfile=./csp/CA_HLL_ROOT_2016/tmp/CA_ISSUER_2016.key \
  --keysize=4096 \
  --digest=sha512 \
  --url=ca.harte-lyne.ca \
  --email=certificates@harte-lyne.ca \
  --csrfile=./csp/CA_HLL_ROOT_2016/csrs/CA_ISSUER_2016.csr \
  "CN=CA_ISSUER_2016,OU=Networked Data Services,O=Harte & Lyne Limited,L=Hamilton,ST=Ontario,C=CA,DC=harte-lyne,DC=ca"
```

## Sign the request.

Because `csp` does not use `openssl ca` to sign requests one can
specify the exact end-date and time in the signing command.

```
csp CA_HLL_ROOT_2016 sign \
  --type=ca  \
  --digest=sha512 \
  --startdate=20161101000000Z \
  --enddate=20351101235959Z \
  --url=ca.harte-lyne.ca \
  --email=certificates@harte-lyne.ca \
  --csrfile=./csp/CA_HLL_ROOT_2016/csrs/CA_HLL_ISSUER_2016.csr
```

## Check that the certificate is in the certs directory

```
ls -l ./csp/CA_HLL_ROOT_2016/certs
total 24
-rw-rw-r--. 1 cspuser cspuser 9556 Nov  1 14:00 01.pem
-rw-rw-r--. 1 cspuser cspuser 9569 Nov  1 15:38 02.pem
```

## Move the new issuing CA's private key

```
mv ./csp/CA_HLL_ROOT_2016/tmp/CA_HLL_ISSUER_2016.key ./csp/CA_HLL_ROOT_2016/private/keys/
```

This is the key which we will use for the issuing CA setup that follows.

## This completes the self-signed root CA creation

# Create the issuing CA

These steps are very similar to those used to create the root CA.

```
csp create

tree csp
csp
├── CA_HLL_ISSUER_2016
│   ├── certs
│   ├── crl_extensions.conf
│   ├── csrs
│   ├── extensions.conf
│   ├── index.txt
│   ├── private
│   │   └── keys
│   ├── public_html
│   │   ├── certs
│   │   │   ├── cert.html.mpp
│   │   │   ├── expired.html.mpp
│   │   │   ├── index.html.mpp
│   │   │   ├── revoked.html.mpp
│   │   │   └── valid.html.mpp
│   │   └── index.html.mpp
│   ├── serial
│   └── tmp
└── CA_HLL_ROOT_2016
    ├── ca.crt
    ├── certs
    │   ├── 01.pem
    │   └── 02.pem
    ├── crl_extensions.conf
    ├── crl-v1.crl
    ├── crl-v1.pem
    ├── crl-v2.crl
    ├── crl-v2.pem
    ├── csrs
    │   ├── ca.csr
    │   └── CA_ISSUER_2016.csr
    ├── extensions.conf
    ├── index.txt
    ├── index.txt.attr
    ├── index.txt.attr.old
    ├── private
    │   ├── ca.key
    │   └── keys
    │       ├── CA_HLL_ISSUER_2016.key
    │       └── CA_HLL_ROOT_2016.key
    ├── public_html
    │   ├── certs
    │   │   ├── cert.html.mpp
    │   │   ├── expired.html.mpp
    │   │   ├── index.html.mpp
    │   │   ├── revoked.html.mpp
    │   │   └── valid.html.mpp
    │   └── index.html.mpp
    ├── serial
    └── tmp
```

## Initialise the new issuing CA using the certificate created above.

```
csp CA_HLL_ISSUER_2016 init --crtfile=./csp/CA_HLL_ROOT_2016/certs/02.pem
```

## Copy the CA's private key from the root CA's key store

The PKI hierarchy desends from the root CA.  So the issing CA's private
key and public certificate must be copied from the root CA store into the
new issuing CA's directory tree.

```
cp from ./csp/CA_HLL_ROOT_2016/private/keys/CA_HLL_ISSUER_2016.key \
        ./csp/CA_HLL_ISSUER_2016/private/ca.key
```

## Check and if necessary modify the permission bits on all the key files to 600.

```
chmod 600 ./csp/CA_HLL_ISSUER_2016/private/ca.key
```

## Copy or modify in place the localised 'extensions.conf' file.

In practice this file will require modification for nearly every
certificate issued.  This is the main reason that CSP is not suited
for a large scale deployment.

```
cp ./extensions.conf.ca_hll_issuer ./csp/CA_HLL_ISSUER_2016/extensions.conf
```
## Generate the crls and public web sites for the new PKI.

```
mkdir -p ./public_html/CA_HLL_ROOT_2016/ ./public_html/CA_HLL_ISSUER_2016/
tree public_html
public_html
├── CA_HLL_ISSUER_2016
└── CA_HLL_ROOT_2016

2 directories, 0 files

for CSPCA in CA_HLL_ROOT_2016 CA_HLL_ISSUER_2016; \
 do  csp $CSPCA gencrl; \
     csp $CSPCA genpublic --export=./public_html/$CSPCA; \
 done;

tree ./public_html
./public_html
├── CA_HLL_ISSUER_2016
│   ├── ca.crt
│   ├── certdb.xml
│   ├── certs
│   │   ├── expired.html
│   │   ├── index.html
│   │   ├── revoked.html
│   │   └── valid.html
│   ├── crl-v1.crl
│   ├── crl-v2.crl
│   └── index.html
└── CA_HLL_ROOT_2016
    ├── ca.crt
    ├── certdb.xml
    ├── certs
    │   ├── 01.crt
    │   ├── 01.html
    │   ├── 01.pem
    │   ├── 02.crt
    │   ├── 02.html
    │   ├── 02.pem
    │   ├── expired.html
    │   ├── index.html
    │   ├── revoked.html
    │   └── valid.html
    ├── crl-v1.crl
    ├── crl-v2.crl
    └── index.html
```

At this point the issuing CA may commence creating end use certificates.
For many private CAs this generally implies that the issuing CA will
create and archive the end use private key as well.

## Example

An employee private key and certificate is created thus:

```
csp CA_HLL_ISSUER_2016 issue \
  --type=user \
  --days=528 \
  --digest=SHA512 \
  --keysize=4096 \
  --email=username@harte-lyne.ca \
  --url=mailto:useremail@harte-lyne.ca \
  "CN=User Name,OU=Employee,O=Harte & Lyne Limited,L=Hamilton,ST=Ontario,C=CA,DC=hamilton,DC=harte-lyne,DC=ca"
```

After each certificate issuing session the CRL and public website
of the issuing CA should be updated.  These should be updated every
thirty (30) days in any case.  Note that the directory for outputting
the html files can be any convenient location.  When the output
location is not specified on the command line `csp` defaults to
`/mnt/floppy`.

```
for CSPCA in CA_HLL_ROOT_2016 CA_HLL_ISSUER_2016; \
 do  csp $CSPCA gencrl; \
     csp $CSPCA genpublic --export=./public_html/$CSPCA; \
 done;
```


