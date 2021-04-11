# stake-pool-setup-guide
Taken from Cardano forums https://forum.cardano.org/t/how-to-set-up-a-pool-in-a-few-minutes-and-register-using-cntools/


Hello everybody,

Have you thinking to run a Cardano POOL but but you don’t have advanced knowledge in linux? No problem! Today I will show you how to setup a Cardano POOL using CNTOOLS in just a few minutes/steps:

But first, few recommendations for security of the nodes (should be applied on all nodes):

create an username/password, change the default ssh connections and deactivate ssh for root user
sudo adduser example 
where example is your new user ; here is recomanded to use a STRONG password; You will use the password to connect to the node via terminal session

add new user in sudo group

sudo adduser example sudo 
test the new user
sudo su - example 
change ssd default port - by default the ssh use port 22, I will recommend to be change, you can choose any port you want.
!!! Before to enable the FIREWALL, add a rule to permit ssh connection on default port (22), otherwise you will loss the connection when you will activate the FIREWALL !!!
-add rule in your firewall for ssh port 22 and activate it

sudo ufw status 
sudo ufw allow proto tcp from any to any port 22
sudo ufw enable

modify the sshd_config file in order to change the ssh port
sudo nano /etc/ssh/sshd_config
#Port 22   
activate the line by deleting # from the beginning and replace 22 with the desired port;

deactivate ssh login for root user
scroll down till you will find the line and modify it to no
PermitRootLogin no 
Ctrl+x y Enter - save the file
add a rule in FIREWALL for the new ssh port configured earlier
sudo ufw allow proto tcp from any to any port x - (the port configured earlier) 
sudo ufw status
restart ssh process and check the status - it should say listening on port xxxx (xxxx - the new port)
sudo systemctl restart ssh
sudo systemctl status ssh
!!! Do not close the initial terminal connection yet, because you will need it in case you can’t access through the new port!

Open a new terminal and try to connect again via ssh but this time use the new port configured above (also try to login with root user , you should not be able to connect anymore)

install google_auth
before to start you will need to install the google_auth app on your mobile; After this step you can continue:
sudo apt install libpam-google-authenticator
!!! the next command will generate a QR code and 5 codes. I recommend you to back-up them somewhere in a safe place, because you will need them in case you will reinstall the google_auth app on another device/mobile. Keep in mind that without google auth you will not be able to connect to the nodes anymore !!!

- google-authenticator - pres Y for all after you bkped the QR and codes
modify the sshd_config file to allow 2Way authentication:
sudo nano /etc/ssh/sshd_config 
ChallengeResponseAuthentication no 
set to yes
ChallengeResponseAuthentication yes
save the file
Ctrl+x y Enter 
restart ssh process
sudo systemctl restart ssh
edit - pam.d/sshd file
sudo nano /etc/pam.d/sshd
add the bellow lines
#One-time authentication via Google Authenticator
auth required pam_google_authenticator.so
save the file
Ctrl+x y Enter 
image

restart ssh process
sudo systemctl restart ssh 
Now you can test if the google_auth is working by connecting again via ssh terminal (after u will enter your ssh password you should see a verification code request: this will be the code generated by your mobile google_auth app)

install fail2ban
sudo apt install fail2ban
check the status of fail2ban, should be running
sudo systemctl status fail2ban 
filter ICMP packets (by enable the filter your nodes will not respond to ping anymore). In Debian-based Linux distributions that ship with UFW application firewall, you can block ICMP messages by adding the following rule to /etc/ufw/before.rules file, as illustrated in the below excerpt.
sudo nano /etc/ufw/before.rules
-A ufw-before-input -p icmp --icmp-type echo-request -j DROP
Ctrl+x y Enter
sudo ufw reload
image

Another great security recommendation 12 comes from @zwirny (thank you)

These are the minimum security steps, you can always improve your security by configuring other steps (like public key, etc), steps which are a little bit harder and you will need to watch video tutorials
==============================================================================

Now, if all went well it’s time to build the node.
Building the NODE

==============================================================================

STEP 1 - downloading the prerequisites:
this step will create the folder structure and it will download the necessary files and scripts to operate the POOL

mkdir "$HOME/tmp"
cd "$HOME/tmp"
curl -sS -o prereqs.sh https://raw.githubusercontent.com/cardano-community/guild-operators/master/scripts/cnode-helper-scripts/prereqs.sh
chmod 755 prereqs.sh
./prereqs.sh 
. "${HOME}/.bashrc"
*check if this command (. “${HOME}/.bashrc”) was configured *
STEP2 - building the node :
this commands will actually build/compile the node and will take around 40-45 minutes.

cd ~/git
git clone https://github.com/input-output-hk/cardano-node
cd cardano-node

git fetch --tags --all
git checkout 1.25.1
git pull origin master

echo -e "package cardano-crypto-praos\n  flags: -external-libsodium-vrf" > cabal.project.local
$CNODE_HOME/scripts/cabal-build-all.sh -o
now you can take a break till the node will be compiled! Enjoy the :coffee:

check the cardano-cli and cardano-node version
cardano-cli --version
cardano-node --version
If everything looks fine go to the next STEP

!!! STEP 2 can be applied with success if you will want to upgrade/downgrade the nodes to a new version; You will only need to replace 1.25.1 with the desired version !!!
==============================================================================

STEP 3 - start and sync the nodes
Once the nodes were built you must start them in order to sync (the nodes will download all the blockchain db locally so it will take more hours)

go to scripts folder :
cd $CNODE_HOME/scripts
or
cd /opt/cardano/cnode/scripts 
open env file and change the CNODE_PORT (by default is 6000 but perhaps you will want to use another ports)
nano env
CNODE_PORT=6000 
save the file
Ctrl+x  y  Enter 
set up the node to start as systemd (this means that if your server will crash or it will reboot, the node/process will start automatically);
!!! remember that all scripts are located inside the scripts folder

./deploy-as-systemd.sh
IMPORTANT to know when you run ./deploy-as-systemd.sh :
It will ask also to set the topologyupdater process as systemd and you will:

for Producer press NO for topologyUpdater

for Relay press YES for topologyUpdater, let the default timer for cnode auto restart to 86400

restart the node and check the status:

sudo systemctl restart cnode
sudo systemctl status cnode
you should see:
image

once the node is up let it sync with the blockchain; you can check the status by running the script:

./gLiveView.sh
You should see the output:

image

after all nodes are 100% synced, it’s time to connect to each other!

==============================================================================

STEP 4 - connect the nodes to each other
Between Producer and Relay will be configured a static connection:

for Producer :
modify the topology file
cd $CNODE_HOME/files
nano topology.json

{
  "Producers": [
    {
      "addr": "RELAY IP ADDRESS",  
      "port": RELAY_PORT,  
      "valency": 1
    }
  ]
}
save the file
Ctrl+x  y  Enter 
*If you have more relays then the topology file must looks like: *

{
  "Producers": [
    {
      "addr": "RELAY1 IP ADDRESS",  
      "port": RELAY_PORT,  
      "valency": 1
    },
      {
      "addr": "RELAY2 IP ADDRESS",  
      "port": RELAY_PORT,  
      "valency": 1
    }
  ]
}
update the FIREWALL rules on Producer to accept connections only from your Relays
sudo ufw allow proto tcp from RELAY_IP to any port xxxx
where xxxx is the Producer CNODE_PORT configured in env file

check if the new line was added in FIREWALL:
sudo ufw status
restart the node wait few minutes an check again with gliveview
cd $CNODE_HOME/scripts
sudo systemctl restart cnode
sudo systemctl status cnode
./gLiveView.sh
in gliveview press P for peer info - check if Relay IP to IN and OUT peers; If all steps went well only the Relay IP should be there

for RELAY - the Relay will connect static with the Producer and dynamically with other public Relays
edit the topologyUpdater script :
cd $CNODE_HOME/scripts
nano topologyUpdater.sh
activate the CUSTOM PEERS line by deleting # from the beginning and add your PRODUCER IP and PORT
CUSTOM_PEERS="1.1.1.1:6000" 
1.1.1.1:6000 is just an example, replace them with your Producer IP/port

also can be added more trusted nodes, separated by | for ex:
CUSTOM_PEERS="1.1.1.1:6000|2.2.2.2:3001|3.3.3.3:1000" 
add a rule in Relay FIREWALL to allow connections from all public Relay on port xxxx ( where xxxx is the CNODE PORT configured in env file)
sudo ufw allow proto tcp from any to any port xxxx
restart the RELAY and check the status in gliveview:
cd $CNODE_HOME/scripts
sudo systemctl restart cnode
sudo systemctl status cnode
./gLiveView,.sh
more infos about topologyUpdater u can find here 22

!!! If everything went well you should see for each node (in ./gLiveView.sh press P for peers) the IP of the remote host to IN and OUT peers.

IF all steps went well the nodes should be connected to each other + the Relay should be connected with other public Relays

We didn’t started the PRODUCER to run as a PRODUCER yet, it will be done after few more steps!

==============================================================================

CREATE THE WALLET AND REGISTER THE POOL via CNTOOLS
ALL BELLOW STEPS WILL BE DONE ON PRODUCER NODE (the NODE which will create BLOCKS)
now we will see the real POWER of CNTOOLS !!! ( thanks to the team who developed)
Actually the CNTOOLS will do the hard job for you AND IT WILL GENERATE ALL THE KEYS, CERTIFICATES, etc which will be stored in priv folder ( path: /opt/cardano/cnode/priv)

REMEMBER: that all scripts are located in scripts folder:
and if we are at this chapter let me show you how the folders tree is built
the folders are located to: /opt/cardano/cnode
here if you press ls -l you should see:

priv folder - where the pool and wallet files like private keys, certificates, metadata, etc are kept
files folder - where the configuration files and topology file, etc are kept
db folder - where the database is kept
socket folder - where the socket-node is kept
scripts folder - where all scripts are kept
==============================================================================

OK, let’s begin:

STEP 5 : - create a new wallet which latter will be associated to the pool (read first till the end before to start)
there are 3 options available

create new wallet
import from mnemonic (restore yoroi or daedalus wallet); Has few limitations such the transactions should be performed from cntools wallet not from Yoroi or Daedalus; but the advantage is that you can restore your wallet with your seed words in case the KEYS files were lost or deleted;
import - HW wallet (available only for local server (the HW wallet should communicate via USB with the server in order to import the KEYS) after you can upload the files to the server and set as a pledge wallet)
My recommendation is to create a new wallet for pool registration and transactions fees only, and pledge with an imported wallet (this way you will have control of your funds better)

1. create a NEW wallet (CAN be used as main WALLET for PLEDGE)

open CNTOOLS
cd $CNODE_HOME/scripts
./cntools.sh  
if you are seeing some infos regarding the last update infos press q and run again

create new wallet - go to wallet and press new or import if u decided to import ur daedalus or yoroi wallet
image

give a name to your wallet and press enter
image

your wallet should be created:
image
image
1277×355 9.83 KB
Copy the address because you will need to send ADA in order to register the pool.

send ADA to wallet (500 ADA + fee for registration)
send the amount of ADA u want to pledge
The “TAX” for POOL registration is 500 ADA, but you will receive them back when you will retire the POOL in the same wallet! Do not delete the wallet till you will not receive them back (2 epoch)!!!

2. Import wallet using mnemonic words (CAN be used as main WALLET for PLEDGE)

go to CNTOOLS - wallet import - mnemonic

import your wallet (edit the wallet name and write the seed words) and read carefully the recommendations:
image
image
1478×673 25.8 KB
if u had funds in your wallet before to import in CNTOOLS, you will need to send ADA again to the CNTOOLS address created (it’s the first daedalus/yoroi address used)

another option is to create in DAEDALUS or YOROI another wallet for POOL pledge, bkp your seed words (must important to not loss), restore in CNTOOLS and after send the pledge to the new address from CNTOOLS (it will be shown also in your daedalus/yoroi wallet)

all future transactions should be performed from cntools not from daedalus or yoroi!!! because if you are doing from daedalus or yoroi the balance will not match and it will show you 0 on cntools
image

if u are receiving the folloing error in CNTOOLS, you will need to rebuild the node; meaning you don’t have cardano-adress and bech32 in ~/.cabal/bin folder:
ERROR: need ‘bech32’ (command not found)
try ‘sudo apt install bech32’
please install with your packet manager of choice(apt/yum etc…) and relaunch CNTools
ERROR: cardano-address and/or bech32 executables not found in path!
Please run updated prereqs.sh and re-build cardano-node

rebuild the node:
cd "$HOME/tmp"
curl -sS -o prereqs.sh https://raw.githubusercontent.com/cardano-community/guild-operators/master/scripts/cnode-helper-scripts/prereqs.sh
chmod 755 prereqs.sh
./prereqs.sh
. "${HOME}/.bashrc"
*check if this command (. “${HOME}/.bashrc”) was configured *

cd ~/git
sudo rm -r cardano-node
git clone https://github.com/input-output-hk/cardano-node
cd cardano-node

git fetch --tags --all
git checkout 1.25.1

echo -e "package cardano-crypto-praos\n  flags: -external-libsodium-vrf" > cabal.project.local
$CNODE_HOME/scripts/cabal-build-all.sh -o
after the node was re-build check if cardano-address and bench32 are present in cd ~/.cabal/bin
folder

cd ~/.cabal/bin
ls -l
you should see:
Mar  1 08:59 bech32
Mar  1 08:59 cardano-address
If you are seeing these 2 files then you rebuild the node succesfully and now you will be able to use import - mnemonic option from CNTOOLS.

3. import wallet from HW wallet (CAN’T be used as main WALLET for PLEDGE)

this must be done locally first and after upload the files to the server

go to CNTOOLS - wallet - import - HW
image

choose your HW type

follow the steps

if all went well your KEYS should be exported/created
image

you will can’t use the HW wallet as a main wallet, you will need to create a new wallet for registration and transactions fees

image

If you will want to ADD/WITHDRAW rewards/SEND funds from the wallet you must go to: CNTOOLS - FUNDS and choose the right operation !

==============================================================================

STEP 6 - create your metadata file and upload it
option 1
create the metadata.json on your PC, using notepad for example, and save the file with .json extension
image
image
1090×770 82.3 KB
store/upload the file in an URL you control. For example https://charity-pool.ro/metadata.json 29

option 2
if you don’t have an URL, you can use a GIST in github
for example https://raw.githubusercontent.com/Alexd1985/a/master/a.json 7

create an account on github 7 and sign in

click new repository
image

type the repository name, click public and crate (you need to use short name to be sure that the link will not be longer than 65 characters)
image

click on creating new file
image

add the file name (short name), edit your metadata informations and press commit new file
image
image
1601×503 29.5 KB
click on branch
image
image
1304×463 28.5 KB
switch the branch to master (click main, type master and click create branch master)
image

click on file and press raw (on the right side)
image

copy the URL and paste it somewhere, we will used later to register the pool
make sure that the URL is less than 65 characters long

a metadata example

{
  "name": "TestPool",
  "description": "The pool that tests all the pools",
  "ticker": "TEST",
  "homepage": "https://teststakepool.com"
  }
==============================================================================

STEP 7 - create the pool
create the pool - inside CNTOOLS go to:
POOL - NEW - type a name for the POOL (this name we will use in Producer env file later, in order to start it like a Producer)
image

you should see:

image

Remember that in order to proceed to the next step you should have 500 + few ADA for fees in your wallet address + the metadata uploaded to your site or github)
you can check the available balance : CNTOOLS - WALLET - SHOW

image
image
1289×284 8.28 KB
==============================================================================

STEP 8 - register the pool
CNTOOLS - POOL - REGISTER - click online
image

select the pool created earlier
image

enter the pledge, margin, cost, metadata, register the relay and choose the wallet from which 500 ADA (tax for registration) will be paid; also set the wallet for pledge (default will be the wallet created earlier but you can choose another wallet if wish but it needs to be created first)
this screenshot is just an example, your inputs can differ:

image
image
1663×998 31 KB
If there are not enough funds in the wallet to pay the transaction (500ADA + fee), the registration process will fail

an example of successfully transaction

image

If you see above message, congrats! your pool should be available on adapools.org 37 or pooltool.io 27

The wallet should have enough founds to cover the PLEDGE declared (+ few ADA for transactions fee) otherwise the PLEDGE will not be MET and you will see this warning on pooltool.io 27 or adapools.org 37
image

If you will want to change the pool’s parameters (like pledge, margin, cost, add/delete relay, etc) go to: CNTOOLS - POOL - MODIFY and it will ask you again to enter the pool’s parameters

==============================================================================

STEP 9 - start your PRODUCER to work as a PRODUCER
after the POOL was registered the env file from your PRODUCER must be updated:

go to scripts folder

cd $CNODE_HOME/scripts 
edit the env file
nano env 
activate the POOL NAME line by deleting # from the beginning and write your pool name
#POOL_NAME=""  
POOL_NAME="test" 
save the file
Ctrl+x y and Enter 
restart cnode wait few minutes and check with gLiveView
sudo systemctl 
./gLiveView.sh 
YOU SHOULD SEE the following output:

image

IF you see informations about KES expiration date, BLOCKS, etc means you configured with success the PRODUCER and now you are ready to create blocks !!!

The CERTIFICATIONS and KES need to be rotated (once/ ~90 days); In order to do that you must go to: CNTOOLS - POOL - ROTATE; after this operation restart your Producer and now in gliveview you should see the new KES expiration date.
==============================================================================

STEP 10 - securing your files
!!! It is NOT recommended to keep the COLD KEYS and PAYMENT KEYS on your live server. Anyway, if you decided to keep them on live server CNTOOLS provides a method to encrypt these files; in this way if some one will hack you and steal the files he can’t use them without the decrypt passord (at least till they will find the password - USE STRONGER PASSWORD)!

!!! BKP your encrypt/decrypt password safe, away from the internet, and be careful to not loss it. If you will loss the password you will can’t decrypt the keys anymore and you will not be able to manage the wallet!!!
My recommendation is to save a copy of files also on an offline device like an USB because the files can be deleted from server by mistake (u can also make a copy on USB before to encrypt the files, this way u will not have problem if u will forget the encryption/decryption password)!!!

(this can be done using filezilla to connect to your node via sftp).

==============================================================================

I told you earlier that your files are store in priv folder… first quit from cntools and type:;

cd $CNODE_HOME/priv
ls -l 
cd wallet - to navigate to wallet folder
cd pool - to navigate to pool folder
cd .. - to navigate back one folder
as u can see the files like cold keys and payment key are not encrypted (not desired - security reason)
image

now go to cntools - and encrypt the files

cd $CNODE_HOME/scripts
./cntools.sh
go to wallet - encrypt - choose the wallet and enter the password

image

to encrypt the pool files go to - POOL - encrypt - choose ur pool - add the password

image

image

Now files are encrypted and even someone will hack your node, he will can’t steal your founds from wallet (at least till they will find the password - USE STRONG password)

!!! Every time when you will want to make a transaction you will need to decrypt the files first !!!

To decrypt the files, follow the above steps but choose DECRYPT instead ENCRYPT.

REMEBER to BKP your password safe, away from the internet, and be careful to not loss it. If you will loss the passwords you will can’t decrypt the keys anymore and you will not be able to managed the wallet !!!
ANOTHER useful option provided for CNTOOLS is to bakp the files (I personally prefer first option):
!!! basically it’s the same thing but a plus is that you have the possibility also to bkp all cnode files; Be carefull when you use this option because at the end it will ask you if you want that the files to be deleted automatically. You can choose NO, and you can delete after manually after you verify that the bkp was created !!!

start CNTOOLS - backup & restore
click backup
choose the directory where the bkp should be stored (type the path like in bellow screenshot and press ENTER)
image
image
1838×963 15.7 KB
choose if you want to bkp all files or just cold keys (choose all files)
image

choose yes to delete the private keys and set an encryption password
image
image
1013×853 19.5 KB
check if your bkp was created:
image

check if your keys were removed :
image

as u can see payment.skey, stake.skey and cold.skey are not present anymore, were deleted.

restore files :

go to CNTOOLS - backup & restore

restore

set the path the bkp file like in bellow screenshot

image
image
1797×930 39.6 KB
enter the restore directory (created if non existent)
image
image
image
1838×963 15.7 KB
enter the passwords
if everything was fine you should see
image

now in your bkp file you have all the keys, keys which must be moved to priv/wallet/test and priv/pool/test folder
image

remember this is an example on an live environment, if you have an offline node you can make the transactions in your offline machine, upload on your live node and then sign the transaction in cntools by choosing transactions - sign (but it’s not the scope of this guide)
!!! KEEP in mind that the steps from this guide will not protect you 100%, you should think and make your OWN PLAN for bkp, storing and restore files !!!
==============================================================================

STEP 11 - download/upload files to server
you can use filezilla to connect via sftp to nodes:

open filezilla - press File - New - New site
enter IP address
enter SSh port
choose PROTOCOL: sftp
LoginMethod - Interactive
enter the user
save and ok
it will ask first for SSH passwords and after for the Google Authenticator (in case u configured)
image
image
1107×978 75.5 KB
===========================

THE END!
IF ALL STEPS WENT WELL your POOL should be visible in adapools.org 37 (yoroi) and pooltool.io 27 (Daedalus)!
I hope this guide will be useful!

Cheers,
