# Deploying an Azure Application Gateway with an existing SSL Certificate from an ARM Template

The [Azure Application Gateway FAQ](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-faq) states that Application Gateways do not integrate natively with Key Vaults. What this appears to mean is that you cannot use [Key Vault Certificates](https://docs.microsoft.com/en-us/rest/api/keyvault/about-keys--secrets-and-certificates#key-vault-certificates) with an Application Gateway, to allow for SSL termination.

While documentation exists for how to upload an existing SSL Certificate to an Application Gateway that has already been created, using either PowerShell or the Azure CLI tools, the documentation that exists on how to create an Application Gateway that performs SSL termination via an ARM Template is not at all clear on how this might be done in a way that allows the SSL certificate information to managed securely.

Here's a solution to that problem that appears to work:

## Convert PEM certificate to PFX

If the existing SSL certificate is in PEM format, it will need to be converted to PFX format first, as Application Gateways require PFX formatted certificates.

```
openssl pkcs12 -export -out certificate.pfx -inkey privateKey.key -in certificate.crt
```

Note that you *must* create the PFX certificate with a password. The Application Gateway definition for ARM Templates appears to *require* that a password for the certificate be supplied when deploying.

## Upload PFX certificate to Key Vault

Rather than uploading the PFX certificate to the Key Vault as a [Certificate](https://docs.microsoft.com/en-us/rest/api/keyvault/about-keys--secrets-and-certificates#key-vault-certificates), instead, the certificate data needs to be uploaded as a [Secret](https://docs.microsoft.com/en-us/rest/api/keyvault/about-keys--secrets-and-certificates#key-vault-secrets). This way, the ARM Template can access the certificate data during deployment of the Application Gateway.

The certificate also needs to be converted into Base64 encoded format, but this can be performed as part of the upload process, using the Azure CLI tool:

```
az keyvault secret set --vault-name KEY_VAULT_NAME --encoding base64 --description text/plain --name SECRET_NAME --file certificate.pfx
```

You should also store the PFX certificate password created as a Secret in the Key Vault as well.

## Reference certificate secret

The PFX certificate data, in Base64 encoded format, and the certificate password, can now both be passed to the ARM Template as a reference (via a parameters file).