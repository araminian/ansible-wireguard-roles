### Requirements

1. Install WireGuard

```bash
ansible-playbook deploy.yml --tags 'wg-install'
```

2. Setup port-forwarding on USG to have SSH access and add server information in `hosts` file. for example: 

```yml
vpn-bnn.zebralog.net ansible_port=31451 ansible_host=87.128.11.69
```
Here we have a VPN server on the Bonn site. We set up port forwarding for SSH access on USG.


3. Add server in `wireguardservers` group.
```yml
[wireguardservers]
vpn-bnn.zebralog.net ansible_port=31451 ansible_host=87.128.11.69
```

### Roles

We have two main roles for two purposes: Add User and Delete User

#### wireguard-add-user

The main jobs of this role are:
- Generate user's public key, private key, and preshared key
- Generate a lock file for each user
- Create peer(client) part of the server config file and store it on ansible controller
- Crete server part of the server config file and store it on ansible controller
- Merge server part and peer parts to generate the final server config file
- Generate client config file

#### wireguard-delete-user

The main jobs of this role are:

- Deactivate user config file
- Move user from active user to deactivate user
- Remove peer part of the server config file that belongs to deactivate user
- Merging server part and peer parts to generate the new server config file

### Variables

#### Users 

1. `users` is a list that shows the active user and you need to add a user to it to create a new user.

2. `deleted-users`  is a list that shows deactivate users. To remove a user you need to move the user from the `users` list to this list. You can remove the user after deleting the user, but keep it for auditing.

#### Network

1. `wireguard_server_public_address` : public address of server. This shows our endpoint. It can be IP or Domain name.

2. `wireguard_interface: wg0` : Specify the WireGuard interface on the server. here is `wg0`.

3. `wireguard_port: 51822` : The port that the server is listening for the incoming connection. If the server is behind the NAT, you should configure port forwarding.

4. `wireguard_server_ip: 10.100.40.1/22` : The server tunnel IP. Clients' gateway. 

5. `wireguard_network_ipv4: 10.100.40.0/22` : The VPN subnet. Clients' will be received an IP from this subnet.

6. `wireguard_dns_servers: 10.100.4.52,10.100.0.52` : The DNS servers that are assigned to the client.

7. `wireguard_allowed_ips: 10.100.0.0/16, 176.9.202.169/32, 138.201.43.66/32, 217.151.152.162/32` : It specifies the networks that should be routed via the tunnel.

#### PKI

1. `wireguard_server_pb_key` : Server Public Key

2. `wireguard_server_pv_key` : Server Private Key

##### How to generate keys?

```
# Generate Private Key
umask 077
wg genkey > privatekey

# Generate Public Key from Private Key
wg pubkey < privatekey > publickey
```

#### Controller Configurations/PKI location

1. `wireguard_local_config_path: "/tmp/wireguard"` : A location on Ansible controller that these file are stored on its sub-folders:
  - `./configs` :  Clients configuration file. you can export config file from this location and send to client.
  - `./pki` : Clients public key, private key and preshared key 
  - `./serverConfigs` : Server and peers configuration parts of the server config file. these files are merged to create a new server config file after adding or deleting users.


> It's recommended to create these folders manually!

#### Server Configurations/PKI location

1. `wireguard_config_path: '/var/wireguard/configs/{{ inventory_hostname }}'` : The location for storing config files and keys.

2. `wireguard_pki_path: "{{ wireguard_config_path }}/pki"` : Here keys are stored. we have two backups of keys. one is here and another is on Ansible controller.

3. `wireguard_activeUser_lockfile_path: "{{ wireguard_config_path }}/active-user/"` : A location for storing lock files for active users.

4. `wireguard_deletedUser_lockfile_path: "{{ wireguard_config_path }}/deleted-user/" ` : The lock files of deactive users are stored here. After deleting user its lock file is moved to this location.