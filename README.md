# Key-Wrapper
Using OpenSSL CHIL Engine with a Thales nCipher HSM to Protect Sensitive Data

The following steps overview a method for "double" encrypting sensitive files with self-signed certificates whose keys are protected in hardware by a Thales nCipher HSM.

### Sample Use Case:
- An organization is operating multiple sites across various geographic regions and needs to securely transmit data from Site X to Site Y. Due to the nature of the data, "out of band" transfer (e.g. physical mail) is required.

### Assumptions:
- HSM security world has been created and has been shared across all involved sites.
- Recipient sites are in possession of a quorum of the HSM operator card set (OCS). 

### General Notes:
- Replace [OCS_NAME] with your target OCS name.
- Replace [DESIRED_CERT_DN_1] with the desired DN for the first self-signed certificate.
- Replace [DESIRED_CERT_DN_2] with the desired DN for the second self-signed certificate.
- The "Data Target Site" is where the data is being transferred to.
- The "Data Source Site" is where the data is being transferred from.

### The Process:

````
1 - Create "Decryption" Private Key #1 (at Data Target Site)
==================================================
# mkdir /home/babcat/key-wrapper
# /opt/nfast/bin/generatekey hwcrhk
protect: Protected by? (token, module) [token] > token
recovery: Key recovery? (yes/no) [yes] > yes
type: Key type? (RSA, DSA, DH) [RSA] > RSA
size: Key size? (bits, minimum 1024) [1024] > 3072
OPTIONAL: pubexp: Public exponent for RSA key (hex)? []
ident: Key identifier? [] > wrappingKey1
nvram: Blob in NVRAM (needs ACS)? (yes/no) [no] > no
key generation parameters:
operation    Operation to perform               generate
application  Application                        hwcrhk
protect
slot
recovery
verify
type
size
pubexp
ident
nvram
Protected by                       token
Slot to read cards from            0
Key recovery                       yes
Verify security of key             yes
Key type                           RSA
Key size                           3072
Public exponent for RSA key (hex)
Key identifier                     wrappingKey1
Blob in NVRAM (needs ACS)          no
Loading `[OCS_NAME]':
Module 1: 0 cards of 1 read
Module 1 slot 0: `[OCS_NAME]' #1
Module 1 slot 0:- passphrase supplied - reading card
Card reading complete.
Key successfully generated.
Path to key: /opt/nfast/kmdata/local/key_hwcrhk_rsa-wrappingKey1


2 -  Create "Decryption" Private Key #2 (at Data Target Site)
==================================================
# /opt/nfast/bin/generatekey hwcrhk
protect: Protected by? (token, module) [token] > token
recovery: Key recovery? (yes/no) [yes] > yes
type: Key type? (RSA, DSA, DH) [RSA] > RSA
size: Key size? (bits, minimum 1024) [1024] > 3072
OPTIONAL: pubexp: Public exponent for RSA key (hex)? []
ident: Key identifier? [] > wrappingKey2
nvram: Blob in NVRAM (needs ACS)? (yes/no) [no] > no
key generation parameters:
operation    Operation to perform               generate
application  Application                        hwcrhk
protect
slot
recovery
verify
type
size
pubexp
ident
nvram
Protected by                       token
Slot to read cards from            0
Key recovery                       yes
Verify security of key             yes
Key type                           RSA
Key size                           3072
Public exponent for RSA key (hex)
Key identifier                     wrappingKey2
Blob in NVRAM (needs ACS)          no
Loading `[OCS_NAME]':
Module 1: 0 cards of 1 read
Module 1 slot 0: `[OCS_NAME]' #1
Module 1 slot 0:- passphrase supplied - reading card
Card reading complete.
Key successfully generated.
Path to key: /opt/nfast/kmdata/local/key_hwcrhk_rsa-wrappingKey2


3 - Create Self Signed Certs (at Data Target Site)
==================================================
# export LD_LIBRARY_PATH=/opt/nfast/toolkits/hwcrhk
# export LIBPATH=/opt/nfast/toolkits/hwcrhk

# /opt/nfast/bin/openssl req -new -sha384 -out /home/babcat/key-wrapper/hsm.wrapper.one.pem -days 365 -x509 -subj "[DESIRED_CERT_DN_1]" -engine chil -keyform engine -key wrappingKey1 -config /opt/nfast/lib/ssleay/openssl.cnf
engine "chil" set.
Enter pass phrase for : [ENTER OCS PASSPHRASE]

# /opt/nfast/bin/openssl req -new -sha384 -out /home/babcat/key-wrapper/hsm.wrapper.two.pem -days 365 -x509 -subj "[DESIRED_CERT_DN_2]" -engine chil -keyform engine -key wrappingKey2 -config /opt/nfast/lib/ssleay/openssl.cnf
engine "chil" set.
Enter pass phrase for : [ENTER OCS PASSPHRASE]


4 - Capture Serial #'s and hash of certificate files (at Data Target Site)
==================================================
# PrettyPrintCert /home/babcat/key-wrapper/hsm.wrapper.one.pem | grep "Serial"
 - Note the "serial #"
# PrettyPrintCert /home/babcat/key-wrapper/hsm.wrapper.two.pem | grep "Serial"
 - Note the "serial #"
# sha256sum /home/babcat/key-wrapper/hsm.wrapper.one.pem
 - Note the sha256 sum
# sha256sum /home/babcat/key-wrapper/hsm.wrapper.two.pem
 - Note the sha256 sum

Communicate both the certificate serial # and sha256sums to the Data Recipient Site.


5 - Receive and Verify Certificate Data (at Data Source Site)
==================================================
- Copy the hsm.transport.one.pem and hsm.transport.two.pem key files to a known location (e.g. /home/babcat)
- Verify certificate data.

  # PrettyPrintCert /home/babcat/hsm.transport.one.pem
   - Note the "serial #" and compare it to that value sent by the Data Target Site.  Ensure values are identical.

  # PrettyPrintCert /home/babcat/hsm.transport.two.pem
   - Note the "serial #" and compare it to that value sent by the Data Target Site.  Ensure values are identical.

  # sha256sum /home/babcat/hsm.transport.one.pem
   - Note the sha256 sum and compare it to that value sent by the Data Target Site.  Ensure values are identical.

  # sha256sum /home/babcat/hsm.transport.two.pem
   - Note the sha256 sum and compare it to that value sent by the Data Target Site.   Ensure values are identical.



6 - Encrypt Data (at Data Source Site)
==================================================
- Assume path of desired encryption package is stored at /home/babcat/datacall.tgz

# openssl smime -encrypt -binary -aes256 -in /home/babcat/datacall.tgz -out /home/babcat/datacall.tgz.enc1 -outform DER /home/babcat/hsm.transport.one.pem

- /home/babcat/datacall.tgz.enc1 will be the encrypted file.  We will now encrypt this again to complete the "double wrap".

# openssl smime -encrypt -binary -aes256 -in /home/babcat/datacall.tgz.enc1 -out /home/babcat/datacall.enc2 -outform DER /home/babcat/hsm.transport.two.pem



7 - Send encrypted data to Data Target Site (at Data Source Site)
==================================================
Burn /home/babcat/datacall.tgz.enc2 and mail USB/CD/DVD to Data Target Site


8 - Decrypt Data (at Data Target Site)
==================================================
- Copy the encrypted data file to /home/babcat

# /opt/nfast/bin/openssl smime -decrypt -binary -in /home/babcat/datacall.tgz.enc2 -inform DER -out /home/babcat/datacall.tgz.enc1 -engine chil -keyform engine -inkey wrappingKey2
engine "chil" set.
Enter pass phrase for : [ENTER OCS PASSPHRASE]

# /opt/nfast/bin/openssl smime -decrypt -binary -in /home/babcat/datacall.tgz.enc1 -inform DER -out /home/babcat/datacall.tgz -engine chil -keyform engine -inkey wrappingKey1
engine "chil" set.
Enter pass phrase for : [ENTER OCS PASSPHRASE]

- The /home/babcat/datacall.tgz will be decrypted.
````


![alt text](https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 1")

