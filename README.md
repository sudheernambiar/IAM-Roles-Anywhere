# IAM-Roles-Anywhere
Create IAM Roles outside outside AWS - more secure than persistent credential storage
## Using OpenSSL CA.
## prerequisites
#### awscli 
```sh
yum install python3-pip -y
pip install awscli
aws --version
               aws-cli/1.29.63 Python/3.9.16 Linux/5.14.0-284.30.1.el9_2.x86_64 botocore/1.31.63
```

#### aws signing helper
```sh
curl -o aws_signing_helper https://rolesanywhere.amazonaws.com/releases/1.1.0/X86_64/Linux/aws_signing_helper
chmod +x aws_signing_helper
./aws_signing_helper version
                              1.1.0
```

## Create a private CA using OpenSSL
#### Generate a private CA key and certificate.
```sh
openssl genrsa -out Private_CA.key 2048
openssl req -new -x509 -days 3560 -key Private_CA.key \
  -subj "/C=IN/ST=New Delhi/L=New Delhi/O=Trustyserves/CN=roles.yourtrust.com" \
  -addext 'keyUsage=critical,keyCertSign,cRLSign,digitalSignature' \
  -addext 'basicConstraints=critical,CA:TRUE' -out Private_CA.crt
```
#### Create client key and generate CSR (create client csr extension file)
```sh
openssl genrsa -out client_private.key 2048
cat > extensions.cnf <<EOF
[v3_ca]
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
EOF
openssl req -new -key client_private.key -subj /CN=app.yourtrust.com  -out client.csr
```
#### Sign the CSR 
```sh
openssl x509 -req -days 3650 -in client.csr -CA Private_CA.crt -CAkey Private_CA.key -set_serial 01 -extfile extensions.cnf -extensions v3_ca -out client_certificate.crt
```

## AWS Colsole we can create a trust anchor first
#### Create a Trust Anchor
In console go to IAM > Roles > Roles Anywhere > Create a trust anchor

give a name(as per your standards) and paste your Private_CA.crt content (Sample Content)
```
-----BEGIN CERTIFICATE-----
MIIDszCCApugAwIBAgIUY8733dMJ2NcVbdeUTuga6w9bJb8wDQYJKoZIhvcNAQEL
BQAwYTELMAkGA1UEBhMCVVMxETAPBgNVBAgMCE5ldyBZb3JrMREwDwYDVQQHDAhO
ZXcgWW9yazEOMAwGA1UECgwFTllVTUMxHDAaBgNVBAMME3JvbGVzLY35Y29tcGFu
eS5jb20wHhcNMjMxMDE2MDQxODU1WhcNMzMwNzE1MDQxODU1WjBhMQswCQYDVQQG
EwJVUzERMA8GA1UECAwITmV3IFlvcmsxETAPBgNVBAcMCE5ldyBZb3JrMQ4wDAYD
VQQKDAVOWVVNQzEcMBoGA1UEAwwTcm9sZXMubXljb21wYW55LmNvbTCCASIwDQYJ
KoZIhvcNAQEBBQADggEPADCCAQoCggEBALujRv2XEP3qh7BUpA6AYXiGqHygfU/E
eV1Q2fFajYH5DtObletXBa4thJlV1d+oTGMJmP/Ut4bd9iLRHhVCt7TRyv67niBR
toeJodeoQZjFhNUn4ShzfbwMTiLPnCqgVPKWtLlMlT7hi7s9P6ZqFYVJL7OEK494
miQpMlkjb5ioVIGzaBSpOoMaQcyEFeDGyU7PVHoCkz67l5M+x09VoZaOIVP2JyZn
sufxFXnq1QJRbjDylgVn/zzajxMS6/xNgmoSuVtoX68eZvN6c2ShH5DPcA5bUx8D
eH0fZAo9MSFDyampuapYlQD/Ln4JsY2TfNJhtq1T7/0Fxiszj8akSRcCAwEAAaNj
MGEwHQYDVR0OBBYEFOBrXe2c00SPhpTZkTgoBeEfewuHMB8GA1UdIwQYMBaAFOBr
Xe2c00SPhpTZkTgoBeEfewuHMA4GA1UdDwEB/wQEAwIBhjAPBgNVHRMBAf8EBTAD
AQH/MA0GCSqGSIb3DQEBCwUAA4IBAQCiAaNcTvG6Nwd0wdeIsnXCkoDEZ2Ep+pTh
jdsBb3g+nFEVuLaa71kcey394/SAZECsqet8uhgo25nR07QkoGhDaZ9rfEsvqXQ/
ya4Ol0dcoJEYArU14GC+37/GIgtslHEhDsWl9oLO1dd2+snm4TkU+eZ6eIQOY6xe
0BMoDUbICiSXXbmYHdXtQyz2UOFx0mJllwv8oPQysSNBzKk+9C3NO4P6KmI+ppt1
ge24SOE/DlwyDj9If26klfBguHraxptyUgjW0ZQ/GcMvnttjyVE+CDuRPLZOORBz
m0NeP3U0v5icjdKXXl4P5uO6euOKrMRIrqK0cBtHAxTU4S1Rgtmv
-----END CERTIFICATE-----
```

#### Create an role anywhere
Create a role with "Custom trust policy"
Under custom trust policy you need to add following
IMP: The CN you added in earlier client certificate csr generation should match with the below, to access the trust.

##### Policy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "rolesanywhere.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession",
                "sts:SetSourceIdentity"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalTag/x509Subject/CN": "app.yourtrust.com"
                },
                "ArnEquals": {
                    "aws:SourceArn": "arn:aws:rolesanywhere:us-east-1:796970074825:trust-anchor/0049afed-5c99-47f5-93b4-9a23dd230f40"
                }
            }
        }
    ]
}
```

you need to provide your trust anchors arn here, instead of 
arn:aws:rolesanywhere:us-east-1:796970074825:trust-anchor/0049afed-5c99-47f5-93b4-9a23dd230f40
also make sure this part matches your cn provided "app.yourtrust.com"

Add needed permissions(permissions to attach) and make sure you have given 
#### Once role is created then we need to additionally add the profile which links the role
```
Go to 
IAM > Roles > Roles > Anywhere > Create a profile
```
In profile you can make your security more robest by addiing additional inline policies.

name yourtrust_profile for the profile(or anything as per standards,)

Once all created now we can go to the client system where all certificate are resides, we are going to use the client_certificate.crt client_private.key along with other arns as per our provisions.

## Get your temporary credentials as per role

```sh
aws configure set credential_process "./aws_signing_helper credential-process  --certificate <client_certificate.crt> --private-key <client_private.key> --trust-anchor-arn <Trust Anchor ARN> \
--profile-arn <Profile ARN> \
--role-arn <Role ARN>"
```
Sample 
```sh
aws configure set credential_process "./aws_signing_helper credential-process  --certificate client_certificate.crt --private-key client_private.key --trust-anchor-arn arn:aws:rolesanywhere:us-east-1:796970074825:trust-anchor/311f8424-5cb5-4f8c-9f72-2a15508a3d2d \
--profile-arn arn:aws:rolesanywhere:us-east-1:796970074825:profile/1df175fe-f08c-4fdf-84d4-bed7e339391f \
--role-arn arn:aws:iam::796970074825:role/jenkins_trust"
```
Once done your .aws/config file will look like below, 
```sh
cat  ~/.aws/config
[default]
credential_process = ./aws_signing_helper credential-process  --certificate client_certificate.crt --private-key client_private.key --trust-anchor-arn arn:aws:rolesanywhere:us-east-1:796970074825:trust-anchor/311f8424-5cb5-4f8c-9f72-2a15508a3d2d --profile-arn arn:aws:rolesanywhere:us-east-1:796970074825:profile/1df175fe-f08c-4fdf-84d4-bed7e339391f --role-arn arn:aws:iam::796970074825:role/jenkins_trust
```

Test sample output, 
aws s3 ls
2023-05-30 12:24:29 mydrums
2023-05-30 17:29:41 mydrums-replica

### Note: Duration of these temp credentials are usually 1 hr, if expires you need to execute the same above credential restore command.
