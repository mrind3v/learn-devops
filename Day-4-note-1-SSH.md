### What is SSH?

SSH or secure shell is a protocol that provides a secure connection between two computers, typically a local computer and a remote server.

It prevents your data on the way to the remote server from being intercepted, read and modified by malicious actors by creating an encrypted tunnel

### How to connect to remote server using SSH?

```sh 
ssh usernameOfRemoteServer@IPAddr/domainNameOfRemoteServer
```
```sh
e.g., ssh mrind3v@192.168.1.1
e.g., ssh mrind3v@server.com
```

It will then prompt you for password if you have not setup keys.
As for keys, you need to have a SSH private key on your local machine and send the public key to remote server

### Setup SSH keys for authorization

First create a private key in your local machine: 
``` sh
ssh-keygen -t rsa -b 4096
```

`-t rsa` specifies the key type (RSA is common).
`-b 4096` specifies the number of bits in the key, making it more secure.

You'll be prompted to "Enter a file in which to save the key". Press Enter to accept the default location (usually `~/.ssh/id_rsa` for the private key and `~/.ssh/id_rsa.pub` for the public key).

You'll then be asked for a "passphrase". This is an optional password to protect your private key. It's recommended for extra security, especially if your private key is ever compromised. If you set one, you'll need to enter it each time you use the key. Press Enter twice if you don't want a passphrase.

Once generated, you'll have two files in your `~/.ssh directory`: your private key (e.g., id_rsa) which stays on your local machine, and your public key (e.g., id_rsa.pub) which you'll send to the server (3:41).

Now copy public key to remote server 

```sh
ssh-copy-id user@remote_host
```

After this step, just do:
```sh 
ssh usernameOfRemoteServer@IPAddr/domainNameOfRemoteServer
```
and it'll give you access to the remote server without prompting for password