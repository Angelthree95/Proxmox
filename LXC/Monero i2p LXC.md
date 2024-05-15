# Deploy a Monero Node available on i2p(d) network on a LXC container
This guide assumes you are runing the system as root user. If not add sudo where needed (almost every command).
I use nano to edit each file, you do you.
This guide can be adapted to a standard debian 11/12 install not specific to Proxmox' LXC.

## The Guide
0. Check for blockchain size in bash: `x=$(curl -sd '{"jsonrpc":"2.0","id":"0","method":"get_info"}' http://node.moneroworld.com:18089/json_rpc | grep 'database_size' |  sed -r 's/.*: ([0-9]*)\,.*/\1/g'); y=$((x/1024/1024/1024)); echo -e "\tCurrent uncompressed Monero block chain is: "$y"GB"`
1. Create new CT with debian (2GB RAM, 1 CPU, 250GB disk or more if blockchain is bigger - leave some space for it to grow)

Skip steps 2-3 if you are not planning on using IPv4 and/or you don't want to route traffic through a VPN enabled router

2. Route all traffic from CT through a VPN connection (example openWRT router, Mullvad WireGuard connection)
3. Check your connection (for Mullvad: `curl https://am.i.mullvad.net/connected` ), install curl if required (`apt update && apt install curl`)
4. Download monero for you system (linux64 in my case): `wget https://downloads.getmonero.org/linux64` (I downloaded mine to '/' not '/root' or user home dir)
4.1. Verify hash validity with values listed in the hashes.txt or monero website: `shasum -a 256 linux64.1`
5. Create destination directory: `mkdir monero`
6. Unpack: `tar -xjvf linux64 -C monero`
6.1 Move contents from a subfolder (mine was monero-x86_64-linux-gnu-v0.18.3.3): `mv /monero/monero-x86_64-linux-gnu-v0.18.3.3/* /monero`
8. Change dir to monero files: `cd monero`
   Steps 8-10 are optional, but highly recommended.
9. Download blochchain for faster import: `wget https://downloads.getmonero.org/blockchain.raw --retry-on-host-error --retry-connrefused --waitretry=1 --read-timeout=20 --timeout=15 -t 0`
10. Import downloaded blockchain: `./monero-blockchain-import --input-file ./blockchain.raw`
11. Delete downloaded blockchain after import: `rm ./blockchain.raw`
12. Run monerod process to see if everything works: `./monerod --detach`. It should be syncing at this point. You should not run monerod as root, but to verify is fine.
13. Tali the log to see what's going on (probably sync): `tail -f ~/.bitmonero/bitmonero.log`. Your log location may very depending on the user you are logged in as.
14. Stop monerod for now: `./monerod exit`
15. Move imported blokchain to new dir: `mv /root/.bitmonero* /monero/.bitmonero/` (if needed, I had to because, I'm using a different user for monerod process)
16. Install requirement for the i2pd part: `apt-get install apt-transport-https`
17. Install i2pd: `wget -q -O - https://repo.i2pd.xyz/.help/add_repo | bash -s -`
17.1 `apt-get update`
17.2 `apt-get install i2pd`
18. Edit monerod config file (or create a new one if you don't have it): `nano /etc/monerod.conf`
```
# Configuration for monerod
# Syntax: any command line option may be specified as 'clioptionname=value'.
#         Boolean options such as 'no-igd' are specified as 'no-igd=1'.
# See 'monerod --help' for all available options.

#1048576 kB/s == 1GB/s; a raise from default 2048 kB/s; contribute more to p2p network
limit-rate-up=1048576
limit-rate-down=1048576

#Limit log size to 5 files 10 MiB each
max-log-file-size=10485760
max-log-files=5
log-file=/var/log/monero/monero.log

#Network settings
p2p-bind-ip=127.0.0.1
p2p-bind-port=18080
rpc-restricted-bind-ip=0.0.0.0
rpc-restricted-bind-port=18089
zmq-rpc-bind-ip=0.0.0.0
zmq-rpc-bind-port=18082
out-peers=64
in-peers=1024

#other settings
disable-rpc-ban=1 #can be helpful if you want to use rpc via i2p
public-node=true #if you want to broadcast your node

```
Ctrl+O, Ctrl+Q (after each nano command)

19. Edit i2pd config: `nano /etc/i2pd/tunnels.conf`
Add at the end of file:
```
[monero-p2p]
type = server
host = 127.0.0.1
port = 8061
inport = 18080
keys = monero-i2pd.dat
```
20. Restart i2pd (reload config): `/etc/init.d/i2pd restart`
21. Lookup your i2pd address: ```curl http://127.0.0.1:7070/?page=i2p_tunnels 2>&1 | grep -Eo "[a-zA-Z0-9./?=_%:-]*" | grep "18083"```. Replace grep port with the port you specified in tunnels.conf.
22. Edit monerod configuration again: `nano /etc/monerod.conf`
Add:
```
#i2pd address
tx-proxy=i2p,127.0.0.1:4447
anonymous-inbound={step 21 address here with .b32.i2p},127.0.0.1:8061
```
23. Create a monerod service: `nano /etc/systemd/system/monerod.service`
```
[Unit]
Description=Monero Node
After=network.target
Wants=network.target

[Service]
User=monero
Group=monero
ExecStart=/monero/monerod --config-file /etc/monerod.conf --non-interactive
Restart=always

[Install]
WantedBy=multi-user.target
```
24. Create new group specified in the monerod.service: `addgroup --system monero`
25. Create new user to run monerod: `adduser --system monero --home /monero`
26. Create logs dir: `mkdir /var/log/monero`
27. Add permission for the user to use logs folder: `chown monero:monero /var/log/monero`
28. Add permission for the monero user to use all files inside /monero dir: `chown -R monero:monero /monero`
29. Restart i2pd (if needed): `/etc/init.d/i2pd restart`
30. Check if i2pd is working fine: `systemctl status i2pd.service`
31. Make sure monero process is stopped: `./monerod exit`
32. Reload services: `systemctl daemon-reload`
33. Start monerod service: `systemctl start monerod.service`
34. Check monerod service status (should be syncing or running if it synced beforehand): `systemctl status monerod.service`
35. You can tail logs using one of theese: `tail -f /var/log/monero/monero.log` or `journalctl -u monerod -f`
36. After syncing run `/monero/monerod print_cn` to see if you are connected to the network. You should be able to see OUT connections. INC will be there in some time (usually up to 12h) if you set your node as public.

## The End
You should be up and running by now. More config options are available at [Monero Wiki](https://getmonero.dev/interacting/monerod).
To write this guide I used:
- [I2P-zero guide by Monero](https://www.getmonero.org/resources/user-guides/node-i2p-zero.html)
- [i2pd documentation](https://i2pd.readthedocs.io/en/latest/)
- [This (and many more) Reddit post](https://www.reddit.com/r/Monero/comments/1ck6b1u/to_public_monero_node_operators_we_need_more/)
- [This Github thread](https://github.com/monero-project/monero/issues/7885)
