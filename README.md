# Key-Wrapper
Using OpenSSL CHIL Engine with a Thales nCipher HSM to Protect Sensitive Data

The following steps overview a method for "double" encrypting sensitive files with self-signed certificates whose keys are protected in hardware by a Thales nCipher HSM.

NOTES:
- Replace [OCS_NAME] with your target OCS name.
- Replace [DESIRED_CERT_DN_1] with the desired DN
