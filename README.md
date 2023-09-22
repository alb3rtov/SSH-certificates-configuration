# How to configure SSH certificates
To configure SSH certificates it is necessary to have the following elements in our scenario:

- Certificate Authority (CA) server
- SSH Host Server
- SSH Client

The CA will be the server is usually managed by a security team. The private keys of the root CA are located on this server and must be protected. If these keys are compromised, it will be necessary to revoke and rotate/recreate all previously generated certificates.

The certificate signed by the host CA is used to prove the host's authenticity to clients. It is sent to the SSH client during the initial handshake when an SSH client attempts to log in.

The SSH client could be a laptop or server of the user running the SSH client. The certificate signed by the client's CA is used to prove the client's authenticity to the host server.

The following will explain the different steps and commands to be followed to configure the different servers and clients for SSH certificate-based authentication.

## Step 1
Generate an SSH key pair from the CA Host (Certificate Authority Host), on the CA server.
```
root@ca-auth# ssh-keygen -t rsa -C HOST-CA -b 4096 -f host-ca
```
The host-ca key is the private key of the host of the certification authority and must be protected. In the event that this key is leaked, all generated certificates should be revoked immediately.

## Step 2
Next, while on the SSH host server, generate a new SSH key pair.
```
root@host-server# ssh-keygen -N '' -C HOST-KEY -t rsa -b 4096 -h -f /etc/ssh/ssh_host_rsa_key
```
By default when openssh-server is installed, a key pair is generated, but they are 2048 bits.

## Step 3
Go back to the CA server and copy the public key generated to the host server:
```
root@ca-auth# scp root@172.17.0.3:/etc/ssh/ssh_host_rsa_key.pub .
```

## Step 4
How generate the certificate from the public key that we have generated and copied in the previous step. For this we are going to use the private key generated in step 1 called host-ca.
```
root@ca-auth# ssh-keygen -s host-ca -I dev_host_server -h -V +52w ssh_host_rsa_key.pub
```
The flags used in the command indicate the following:
- -s host-ca: specifies the private key file of the certificate authority used for signing.
- -I dev_host_server: indicates the identity of the certificate. This value can also be used to revoke the certificate in the future.
- -h: the certificate will be a host certificate instead of a user certificate
- -V: indicates the validity period of the certificate. By default certificates are valid forever by default, so it is recommended to set this value, for example, to 52 weeks (1 year).

## Step 5
Once the certificate is generated, copy it to the SSH host server.
```
root@ca-auth# scp ssh_host_rsa_key-cert.pub root@172.17.0.3:/etc/ssh/ssh_host_rsa_key-cert.pub
```

Remove the obsolete public key and the host certificate from the CA server.
```
root@ca-auth# rm ssh_host_rsa_key-cert.pub ssh_host_rsa_key.pub
```




