# Automation for creating a certificate with a CA not partnered with Key Vault

These instructions assume a Key Vault exists, with a pending certificate request to allow download of the CSR from Key Vault, signed by the CA, and the resulting certificate merged back into Key Vault.  This mirrors the flow outlined in the Azure documentation for the scenario: [Creating a certificate with a CA not partnered with Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/certificates/certificate-scenarios#creating-a-certificate-with-a-ca-not-partnered-with-key-vault).

The instructions below can be followed using the [Azure Cloud Shell in bash mode](https://learn.microsoft.com/en-us/azure/cloud-shell/quickstart?tabs=azurecli), removing the need for local computer configuration and tools.


## Create the Base Infrastructure
Execute the Terraform in `main.tf` to create the infrastructure: Resource Group, Key Vault, Role Assignment, Certificate.  Feel free to skip the Terraform and do this manually via the Azure Portal.

## Create a Self-Signed Root CA
```
openssl req -x509 -sha256 -days 1825 -newkey rsa:2048 -keyout rootCA.key -out rootCA.crt
```

## Download the CSR using the Azure CLI
Using the [az cli certificate pending show](https://learn.microsoft.com/en-us/cli/azure/keyvault/certificate/pending?view=azure-cli-latest#az-keyvault-certificate-pending-show) command:
```
az keyvault certificate pending show --name cert --vault-name THE-NAME-OF-YOUR-KEYVAULT
```

Save the CSR property to a new file named `your.csr`, with a `BEGIN CERTIFICATE REQUEST` header and `END CERTIFICATE REQUEST` trailer, e.g.
```
BEGIN CERTIFICATE REQUEST
... the 'csr' value from the az cli command ...
END CERTIFICATE REQUEST
```

## Sign the CSR With the Root CA
```
openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -in your.csr -out your.crt -days 365
```

A file called `your.crt` should now exist, signed by the self-signed root CA.

## Merge the Certificate into Key Vault
Using the [az keyvault certificate pending merge](https://learn.microsoft.com/en-us/cli/azure/keyvault/certificate/pending?view=azure-cli-latest#az-keyvault-certificate-pending-merge) command:
```
az keyvault certificate pending merge --file your.crt --name cert --vault-name THE-NAME-OF-YOUR-KEYVAULT
```

The certificate in Key Vault will now be in a 'Completed' state.

# Terraform AzAPI
As an alternative to using `az cli` for the `show` and `merge` operations it may be possible to continue using Terraform but with the [AzAPI Provider](https://learn.microsoft.com/en-us/azure/developer/terraform/overview-azapi-provider) to access the Key Vault REST APIs for [GetCertificateOperation](https://learn.microsoft.com/en-us/rest/api/keyvault/certificates/get-certificate-operation/get-certificate-operation?tabs=HTTP#getcertificateoperation) and [MergeCertificate](https://learn.microsoft.com/en-us/rest/api/keyvault/certificates/merge-certificate/merge-certificate?tabs=HTTP) instead.  This would need exploring and to understand the Terraform lifecycle throughout the async/long certificate process.