# :page_with_curl: How to configure SSH certificates :page_with_curl:
## Scenario
To configure SSH certificates it is necessary to have the following elements in our scenario:

- Certificate Authority (CA) server
- SSH Host Server
- SSH Client

The CA will be the server is usually managed by a security team. The private keys of the root CA are located on this server and must be protected. If these keys are compromised, it will be necessary to revoke and rotate/recreate all previously generated certificates.

The certificate signed by the host CA is used to prove the host's authenticity to clients. It is sent to the SSH client during the initial handshake when an SSH client attempts to log in.

The SSH client could be a laptop or server of the user running the SSH client. The certificate signed by the client's CA is used to prove the client's authenticity to the host server.

The following will explain the different steps and commands to be followed to configure the different servers and clients for SSH certificate-based authentication.

## Enviroment setup
For ease of configuration, I have created a `Dockerfile` with the necessary containers for the scenario configuration. The necessary hostnames, services and users are created and configured automatically. The configuration to be carried out is the one below.

To automate the deployment of the environment use `Makefile` as follows:
```
command
```

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
Now you have to generate the certificate from the public key generated and copied in the previous step. To do this, use the private key generated in step 1 called host-ca.
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

## Step 6
Now from the SSH host server the copied certificate must be configured. To do this, add it to the configuration file located in the path `/etc/ssh/sshd_config`.
```
root@host-server# echo 'HostCertificate /etc/ssh/ssh_host_rsa_key-cert.pub' | tee -a /etc/ssh/sshd_config
```

And restart the SSH service.
```
root@host-server# service ssh restart # Ubuntu/Debian
```

## Step 7
To configure the SSH client, the public key of the certification authority is required, as it must be stored in the /.ssh/known_hosts file of the clients to be configured. For example, you can first copy the public key to the client and then configure it.

Now from the SSH client, you have to configure the public key as a certification authority. The "\*" indicates the list of servers that have signed their host key. In the case of "\*", it indicates that you allow all servers.
```
root@client# echo ‘@cert-authority * clave-publica-host-ca’ | tee -a ~/.ssh/known_hosts
```

## Step 8
A key pair must be generated on the certification authority's server, which will be used to sign the user's certificates.
```
root@ca-auth# ssh-keygen -t rsa -C CLIENT-CA -b 4096 -f client-ca
```

## Step 9
Copy the public key generated in the previous step to the SSH host server **(NOT to the clients)**.
```
root@ca-auth# scp client-ca.pub root@172.17.0.3:/etc/ssh/client-ca.pub
```

## Step 10
From the SSH host server, the copied public key must be configured by adding it to the SSH configuration file located in `/etc/ssh/sshd_config`.
```
root@host-server# echo 'TrustedUserCAKeys /etc/ssh/client-ca.pub' | tee -a /etc/ssh/sshd_config
```

And restart the SSH service.
```
root@host-server# service ssh restart # Ubuntu/Debian
```

## Step 11
To test the operation, a key pair must be generated in the SSH client, setting a passphrase and a name for the public and private key.
```
test@client:$ ssh-keygen -t rsa -b 4096
```

The public key must be copied to the CA server.
```
test@client:$ scp ~/.ssh/id_rsa.pub root@172.17.0.2:/id_rsa.pub
```

It is necessary to sign with the private key of the users' CA to generate the certificate. In this case the -n option must be added to indicate a list of users with whom the certificate will be valid to authenticate.
```
root@ca-auth# ssh-keygen -s client-ca -I test -n root,test -V +52w -z 1 id_rsa.pub
```

## Step 12
The certificate generated in the previous step must be copied and stored in the user's `/.ssh` directory.
```
root@ca-auth# scp id_rsa-cert.pub test@172.17.0.4:/home/test/.ssh/id_rsa-cert.pub
```

Finally, it is necessary to test the operation using the SSH client. From the /.ssh directory, try to access the SSH host server.
```
test@client:$ ssh -v root@172.17.0.3

test@client:$ ssh -v test@172.17.0.3
```
