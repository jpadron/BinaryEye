# Generar certificado CA propio para BinaryEye

## 1. Generar la CA (una sola vez)

### OpenSSL

```bash
openssl genrsa -out miCA.key 2048

openssl req -x509 -new -key miCA.key -sha256 -days 3650 -out miCA.crt -subj "/CN=Mi CA Interna"
```

### PowerShell

```powershell
$ca = New-SelfSignedCertificate -Subject "CN=Mi CA Interna" -CertStoreLocation "Cert:\CurrentUser\My" -KeyUsage CertSign, CRLSign -KeyLength 2048 -HashAlgorithm SHA256 -NotAfter (Get-Date).AddYears(10) -TextExtension @("2.5.29.19={critical}{text}ca=TRUE")

Export-PfxCertificate -Cert $ca -FilePath "miCA.pfx" -Password (ConvertTo-SecureString -String "password" -Force -AsPlainText)

Export-Certificate -Cert $ca -FilePath "miCA.crt" -Type CERT
```

## 2. Generar el certificado del servidor firmado por la CA

### OpenSSL

```bash
openssl genrsa -out servidor.key 2048

openssl req -new -key servidor.key -out servidor.csr -subj "/CN=miservidor.local"

openssl x509 -req -in servidor.csr -CA miCA.crt -CAkey miCA.key -CAcreateserial -out servidor.crt -days 730 -sha256
```

### PowerShell

```powershell
$ca = Get-ChildItem -Path "Cert:\CurrentUser\My" | Where-Object { $_.Subject -eq "CN=Mi CA Interna" }

$servidor = New-SelfSignedCertificate -Subject "CN=miservidor.local" -CertStoreLocation "Cert:\CurrentUser\My" -Signer $ca -KeyLength 2048 -HashAlgorithm SHA256 -NotAfter (Get-Date).AddYears(2) -TextExtension @("2.5.29.17={text}DNS=miservidor.local")

Export-PfxCertificate -Cert $servidor -FilePath "servidor.pfx" -Password (ConvertTo-SecureString -String "password" -Force -AsPlainText)
```

## 3. Uso

| Archivo | Dónde se usa |
|---------|-------------|
| `servidor.crt` + `servidor.key` (o `servidor.pfx`) | En el servidor web |
| `miCA.crt` (o `miCA.pfx`) | Instalar en el tablet Android como certificado CA |

### Instalar CA en Android

1. Copiar `miCA.crt` al tablet
2. Ajustes > Seguridad > Instalar desde almacenamiento
3. Seleccionar el archivo y elegir "CA" como tipo

Con la CA instalada en el dispositivo, el `network_security_config.xml` (que ya incluye `<certificates src="user" />`) permitirá la conexión HTTPS sin necesidad del trust-all en código.
