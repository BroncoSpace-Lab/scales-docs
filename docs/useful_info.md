# Useful Information and Commands for SCALES

## IMX8X

We are actually using an IMX8QuadXPlus which will make a difference in the development of the operating system.

Command to ssh into the IMX:

```

ssh root@<ip address> -o HostKeyAlgorithms=+ssh-rsa -o PubKeyAcceptedAlgorithms=+ssh-rsa

```

Copy files from one computer to the IMX:

```

cd <file directory>
scp -o HostKeyAlgorithms=+ssh-rsa -o PubKeyAcceptedAlgorithms=+ssh-rsa <file name> root@<ip address>:~

```
