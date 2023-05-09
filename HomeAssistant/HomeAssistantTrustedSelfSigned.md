
# HomeAssistant: Trusted Self-Signed Certificates (HA Mobile App + Browser)

#### This is a quick guide on creating self-signed IP-backed certificates for your local HomeAssistant server that are validated by the HA Mobile App and can also be added to browsers.

##### Note that the general guidance is to generate valid domain-backed certificates and regularly update them using Let's Encrypt so use this guide as a last resort.

----


1) Create your own CA
---

On your PC use your already-installed OpenSSL (if not, install it using your preferred distro's package manager).

This will create your root key, so make sure the directory you're working in is not world readable and the PC you're using is somewhat trustworthy.

You will need to chose a CN/Organization Name for your Root CA and for your subsequent certificates.

First, generate key curve parameters, as a curve reference for later OpenSSL commands:

```
$ openssl ecparam -out ecca.param -name secp384r1
```

Then generate your CA's EC Root (self-signed) key:
```
$ openssl req -nodes -newkey ec:ecca.param -days 3650 -x509 -sha256 -keyout ecca.key -out ecca.crt
```

This will create both a `.key` and `.crt` file in the current directory.

You can also combine your Root CA key & self-signed certificate in a `.pem` format, but it's not necessarily mandatory :

```
$ cat ecca.crt ecca.key > ecca.pem
```

Make sure these files are only readable by your user (`chmod 600 ecca*`).

2) Configure the OpenSSL environment
---

After that, let's prepare the Demo CA directory hierarchy so that OpenSSL knows where to put its index/revocation for generated certificates and initialize the serial number to 01 for the first signed certificate :

```
$ mkdir ./demoCA/
$ mkdir ./demoCA/certs
$ mkdir ./demoCA/crl
$ mkdir ./demoCA/newcerts
$ mkdir ./demoCA/private
$ touch ./demoCA/index.txt
$ echo 01 > ./demoCA/serial
```

Then, **before** creating your HA Server's key, you will need to generate a global OpenSSL configuration file that will modify OpenSSL's behaviour for multiple commands (`ca`, `req`).

Here is a suggested configuration (beware of the `<PLACEHOLDERS>` you need to replace to match your own environment). This configuration was adapted from [https://jamielinux.com/docs/openssl-certificate-authority/appendix/root-configuration-file.html](https://jamielinux.com/docs/openssl-certificate-authority/appendix/root-configuration-file.html).

```
# OpenSSL root CA configuration file.
# Copy to <YOUR_WORKING_DIR>

[ ca ]
# see `man ca` for options
default_ca = CA_default

[ CA_default ]
# Directory and file locations for your Demo CA to track certificates issued/revoked.
dir               = <PATH_TO_DEMOCA_DIR>
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate, <REFER> to the ones you just created.
private_key       = $dir/ecca.key
certificate       = $dir/ecca.crt

# Here you could consider adding CRL (revocation) configurations as well.
#crlnumber         = $dir/crlnumber
#crl               = $dir/crl/ca.crl.pem
#crl_extensions    = crl_ext
#default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-512 instead.
default_md        = sha512

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 3650 # <MODIFY> to fit your preferences
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
# Ensure your usage of the `ca` command adds the right SAN
subjectAltName = @alt_names

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

[ req ]
# Configuration for the `req` command used later
default_md = sha512
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
# Note that more optional fields exist for this block (locality, etc.)
C = <YOUR_COUNTRY_NAME>
ST = <YOUR_STATE_NAME>
O = <YOUR_ORG_NAME_MUST_MATCH_CA>
CN = <YOUR_SERVER_IP>

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
# Refer to the SAN configured hereafter
subjectAltName = @alt_names

[req_ext]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = <YOUR_SERVER_IP>

```

You can save this file as `ssl.cnf` which we will refer to from now on.


3) Generate your server certificate signature request (CSR)
---

Now we'll generate your HA Server's CSR ensuring that the configuration is sent as a command parameter:

```
openssl req -nodes -newkey ec:ecca.crt -sha512 -keyout server.key -out server.csr -config ssl.cnf
```

This should create the `server.key` and `server.csr` file in the current directory.
You should validate that the CSR has the right `Subject Alternative Name` field using the following command:

```
openssl req -text -noout -verify -in server.csr
```

This should output your certificates detail but do validate the following exists (and matches the `CN = <YOUR_SERVER_IP>`).

```
        Requested Extensions:
            X509v3 Subject Alternative Name:
                IP Address:<YOUR_SERVER_IP>
```

4) Sign your server certificate
---

Now, let's sign your CSR using your pre-created Root CA using the following command:

```
openssl ca -extensions v3_ca -days 3650 -out server.crt -in server.csr -cert ecca.crt -keyfile ecca.key -config ssl.cnf
```

This command should output the following, do validate one more time that the `commonName` and the `Subject Alternative Name` fields exist and correlate.

```
Using configuration from ssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:

...

            commonName                = <YOUR_SERVER_IP>
        X509v3 extensions:

...

            X509v3 Subject Alternative Name:
                IP Address:<YOUR_SERVER_IP>

Certificate is to be certified until ...
Sign the certificate? [y/n]:y

1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Congrats! You're now in posession of a certificate that *could* be trusted by Android & your HA mobile App, provided the Root CA is added to the trusted root CAs. So let's do that.

5) Upload your server certificate your HA config directory
---
Now you can `scp` your `.crt` and your `.key` file to your HA `/config` directory (use HA SSH add-ons to connect if you're using HA OS). Make sure they are only accessible by `root` (or the user running HA).

Your `configuration.yaml` file should refer to these newly created certificates:

```
http:
  ssl_certificate: /config/server.crt
  ssl_key: /config/server.key
```

Once the files are there and configured, do restart HomeAssistant using `Developer Tools->Check Configuration` followed by `Developer Tools->Restart->Restart Home Assistant`.

Since you modified the certificate, it is possible that your browser cache needs to be cleared.

6) Add your Root CA to Android Trusted CAs
---

Send your `ecca.crt` file to your mobile phone (preferably using a direct USB connection).

On your mobile device (Example here for Android 13), navigate to your `settings` menu.

Then go to `Security & privacy -> More security settings -> Encryption & credentials -> Install a Certificate`.

Chose `CA certificate`, confirm that you want to `Install anyway`, enter your device lock code, and chose the `ecca.crt` file you just sent to your phone.

To validate proper installation, going back to the `Encryption & credentials` settings menu, click on the `Trusted credentials` then select the `User` tab at the top and you should see your Root CA certificated added here.

7) *Optional* Add your Trusted Root CA to your Browser Certificate Store
---

Follow any of the online guides for browsers e.g. [Firefox](https://google.gprivate.com/search.php?search?q=add+root+certificate+to+mozilla+firefox).


8) Reload the Android HA Mobile App
---

On your mobile device, do reload the HA App and add a new server using your URL at the IP you requested a certificate for, e.g. `https://<IP>:PORT` and if you've done everything right, the app should not complain.

You can test beforehand using Android Chrome since it's leveraging the same local certificate store, it should not complain that something's wrong with the CN, CA, or anything else.
