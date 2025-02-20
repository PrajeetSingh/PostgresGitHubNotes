## Create Password-less authentication from Primary to Standby Server
```sh
ssh-keygen -t rsa
cd .ssh
cat id_rsa.pub

# Copy above public key into Standby/Replica server in file ~/.ssh/authorized_keys

# If you want to do it from Standby to Primary too, then run above commands on Standby database, 
# then copy the content of id_rsa.pub from Standby to Primary server in ~/.ssh/authorized_keys.
```