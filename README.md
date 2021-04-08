# pki-secrets-engine
https://www.vaultproject.io/docs/secrets/pki


vault server -dev

export VAULT_ADDR="http://127.0.0.1:8200"

vault login <rook_token>

Mount the backend is done with:
vault secrets enable pki


Configure CA certificate - issued like directly from the root
vault secrets tune -max-lease-ttl=87600h pki

Generate root certificate
vault write pki/root/generate/internal common_name=myvault.com ttl=87600h

Setup the URL configuration
vault write pki/config/urls issuing_certificates="http://vault.example.com:8200/v1/pki/ca" crl_distribution_points="http://vault.example.com:8200/v1/pki/crl"

Configure a role
vault write pki/roles/example-dot-com \
    allowed_domains=example.com \
    allow_subdomains=true max_ttl=72h


Issue certificates
vault write pki/issue/example-dot-com \
    common_name=blah.example.com


Setting Up Intermediate CA

This guide builds on the previous guide's root certificate authority and creates 
an intermediate authority using the root authority to sign the intermediate's certificate.

Mount the backend
Note: mount it at a different path 
vault secrets enable -path=pki_int pki

Configure an Intermediate CA
That sets the maximum TTL for secrets issued from the mount to 5 years. 
This value should be less than or equal to the root certificate authority.
vault secrets tune -max-lease-ttl=43800h pki_int

 generate our intermediate certificate signing request:
 vault write pki_int/intermediate/generate/internal common_name="myvault.com Intermediate Authority" ttl=43800h

 Take the signing request from the intermediate authority and sign it using another certificate authority, 
 in this case the root certificate authority generated in the first example.
 vault write pki/root/sign-intermediate csr=@pki_int.csr format=pem_bundle ttl=43800h

 echo "-----BEGIN CERTIFICATE REQUEST-----
MIICcjCCAVoCAQAwLTErMCkGA1UEAxMibXl2YXVsdC5jb20gSW50ZXJtZWRpYXRl
IEF1dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMXLqh+K
t7xNN4aX+G+Q4KOwEkuxV1YwaDNGTTfuMKyhrCsyZQ562rIEoUgP1DsLQKFgIWNN
abo16BEGKSIf1nmSKyV58ZxWRbzRlkJCxitqUCPUxAOYIygUPeTEJZOAT8LJ0GIw
SIUCNcj3dwrvVdDs/jVUZOBy6tPmKjdKCVoBob9KaSF6Gf0Sg1ci6swJWRpBITdL
d99SkdKTFRfatp5VCqKAUFaENmCXZv+/vBsBrqyduLTVCzWPX1SwnF50g30ZxnYu
u/zBMtK1AIfnqt5EOybaAnMlDTf6YqhVVBN85hLSsmHuQVb4cQyWor440rUxOB/I
sXfYdZBsbTK7oI0CAwEAAaAAMA0GCSqGSIb3DQEBCwUAA4IBAQBN26uKF80L3omS
vxuiTqtW3kZsNmSCwagxmFV1q1wxljwxO9YS85aTPWuA5FMLTl67K92+wJ+QKFlj
jXDFjoaFdCtNJa3dsaQMNyfltDgxEhsjoQB/xNN/oUzY5VUWrgfnSuTpE0lotzEN
KxmHcQE7RtwyHz+Cs5ziwkc+oTki+YcZcCKJ6b5BGAL4mNvpz1iNszf1L9pbly/C
6x0u6wbz1hZzsASAECof66K3REszLV7sM9OA9+S1VsONCW6kx30d28W4iXeDdLBV
UNT9y/N3ZvBY0/gsJ2qKmsDYg/zZQvQvREGJiGx4EnBMjAcQrecXURYMgNVRPgxk
Vhvz4qbV
-----END CERTIFICATE REQUEST-----" > pki_int.csr

vault write pki/root/sign-intermediate csr=@pki_int.csr format=pem_bundle ttl=43800h


Now set the intermediate certificate authorities signing certificate to the root-signed certificate.
echo "-----BEGIN CERTIFICATE-----
MIIDuTCCAqGgAwIBAgIUFekbeXeEwdTKI31PQskFTi23qmkwDQYJKoZIhvcNAQEL
BQAwGTEXMBUGA1UEAxMObXktd2Vic2l0ZS5jb20wHhcNMjEwNDA4MDkzNTQ4WhcN
MjYwNDA3MDkzNjE4WjAtMSswKQYDVQQDEyJteXZhdWx0LmNvbSBJbnRlcm1lZGlh
dGUgQXV0aG9yaXR5MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxcuq
H4q3vE03hpf4b5Dgo7ASS7FXVjBoM0ZNN+4wrKGsKzJlDnrasgShSA/UOwtAoWAh
Y01pujXoEQYpIh/WeZIrJXnxnFZFvNGWQkLGK2pQI9TEA5gjKBQ95MQlk4BPwsnQ
YjBIhQI1yPd3Cu9V0Oz+NVRk4HLq0+YqN0oJWgGhv0ppIXoZ/RKDVyLqzAlZGkEh
N0t331KR0pMVF9q2nlUKooBQVoQ2YJdm/7+8GwGurJ24tNULNY9fVLCcXnSDfRnG
di67/MEy0rUAh+eq3kQ7JtoCcyUNN/piqFVUE3zmEtKyYe5BVvhxDJaivjjStTE4
H8ixd9h1kGxtMrugjQIDAQABo4HkMIHhMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMB
Af8EBTADAQH/MB0GA1UdDgQWBBRyyRmV81DSCajr+3rQ321/c4QeyDAfBgNVHSME
GDAWgBSlpXy70IdCFtPX6VFHCSoy5SWAWzBDBggrBgEFBQcBAQQ3MDUwMwYIKwYB
BQUHMAKGJ2h0dHA6Ly92YXVsdC5leGFtcGxlLmNvbTo4MjAwL3YxL3BraS9jYTA5
BgNVHR8EMjAwMC6gLKAqhihodHRwOi8vdmF1bHQuZXhhbXBsZS5jb206ODIwMC92
MS9wa2kvY3JsMA0GCSqGSIb3DQEBCwUAA4IBAQCR6QeSs52RkI32kk8N1hdsm4xC
Fgi5+b4Kw66nY7X37Aj7nWtf8T8eI2yrwRPdMhDtsuDB3IgR5vDsYm594o9GexYk
HKrsN4UvvwDhxgUi+ghQGaxpFsM7xTEKBLdgBnkPhMRuIJGETXer/HIiygBMRINh
KDPAovC82XAkKstIxFt14PMfIg0vP0KH0iiYTcjYirlMowbWoyUZ3Is3KkkAuE7n
AWEFKzaamq3xvM8X6yiBicIeFz3aBhccfI9LahoN+v2v7g34ku4OuoiQf0cvEhR9
MRjdHUzLH7JpmyXo0Xosnb1qavmZFvkWkgqtC0BSNz9qh1Y0b5TpNTpjKlxR
-----END CERTIFICATE-----" > signed_certificate.pem

vault write pki_int/intermediate/set-signed certificate=@signed_certificate.pem

Set URL configuration
vault write pki_int/config/urls issuing_certificates="http://127.0.0.1:8200/v1/pki_int/ca" crl_distribution_points="http://127.0.0.1:8200/v1/pki_int/crl"

Configure a role
A role is a logical name that maps to a policy used to generate those credentials. 
vault write pki_int/roles/example-dot-com \
    allowed_domains=example.com \
    allow_subdomains=true max_ttl=72h


Issue Certificates
vault write pki_int/issue/example-dot-com \
    common_name=blah.example.com
