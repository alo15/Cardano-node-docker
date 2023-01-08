# Cardano Node Docker Image (for ARM64 devices) 🐳

In this project you will find the files to build a docker image on Linux containing all the needed files to run a Cardano full node.
The docker image can run on any arm64 device (such as a RaspberryPi, Mac Mini M1, etc.). It can be configured as a relay or block production node.

If you are enjoying the content of this project, please consider supporting me by donating ₳D₳ to: addr1qygv5fqsfjhfgkx7fhkkegxksx56dsu262vhaxr4mvuukt8uqh7nhjs3pcl98xr2zhmtqk6qkmr4gszxjrs3lnpedqdqyr3jzc

## Why using docker image to run a Cardano node?

The elegant thing of a Cardano node deployed as a Docker image is, that it can be installed and started seamlessly straight out-of-the box.
The day you should decide to delete it, you just have to remove one file - the image. Another advantage is that it can run on any operating 
system that has Docker installed. With a Docker image the setup and management of a Cardano node is easier compared to the traditional guide 
(for example you don't have to deal with systemd settings). It is therefore recommended also for less experienced users.

## System requirements

* CPU: ARM64 processor min 2 cores at 2GHz or faster.
* Memory: 16GB of RAM.
* Storage: 50 GB.
* OS: Linux (recommended Ubuntu)
* Additional Software: Docker
* Broadband: 10 Mbps +

If you intend to use a Raspberry Pi 8GB RAM for the deployment of this docker image, I highly recommend to follow the Armada Alliance 
[Server Setup guide](https://armada-alliance.com/docs/stake-pool-guides/pi-pool-tutorial/pi-node-full-guide/server-setup/) first. 
This guide describes how to optimize the Hardware in order to meet the above listed system requirements.  

# 1.Install Docker

The installation of Docker varies from operating system to operating system. For this reason, I share some helpful and good installation 
guide links for the different operating systems.

* [Install Docker on Linux (Ubuntu)](https://github.com/speedwing/cardano-staking-pool-edu/blob/master/DOCKER.md)
* [Install Docker on MacOs](https://docs.docker.com/desktop/mac/install/)
* [Install Docker on Windows](https://docs.docker.com/desktop/windows/install/)

# 2. Download repository and Cardano node configuration files

Let's clone this repository to your host system first

```bash
cd ${HOME}
sudo git clone https://github.com/jterrier84/Cardano-node-docker.git
cd Cardano-node-docker
```  

We will now download the latest official Cardano node configuration files from the IOHK repository and store them on our host system.

For the sake of this tutorial we will download and set up the configuration files for the Cardano testnet. If you need the files for the mainnet
just replace "testnet" with "mainnet" here below.
 
Note: As the configuration files might require modifications over time, it is way more practical to have them stored on the host, 
rather than have them stored inside the Docker container. The Docker image will then access to these files via file sharing.

```bash
sudo mkdir node/db
sudo mkdir node/files
cd node/files
export NODE_CONFIG="testnet"
export NODE_BUILD_NUM=$(curl https://hydra.iohk.io/job/Cardano/iohk-nix/cardano-deployment/latest-finished/download/1/index.html | grep -e "build" | sed 's/.*build\/\([0-9]*\)\/download.*/\1/g') 
sudo wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-config.json
sudo wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-byron-genesis.json
sudo wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-shelley-genesis.json
sudo wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-alonzo-genesis.json
sudo wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-topology.json
sudo wget -O tx-submit-mainnet-config.yaml https://raw.githubusercontent.com/input-output-hk/cardano-node/master/cardano-submit-api/config/tx-submit-mainnet-config.yaml
```

Note: The Docker Cardano node will access to the directories /files and /db on the host system. 

The directory /files contains the downloaded Cardano node configuration files.

The /db directory will host the Cardano blockchain once the Docker node is started. It is important that the blockchain data 
are stored on the host system and not inside the Docker container, otherwise the entire blockchain would be deleted every time 
the Docker container is removed. Our docker image will manage this automatically.

# 3. Build the Cardano node docker image

At this point it's time to build the docker image. The image will include:

1. cardano-node & cardano-cli - Cardano binaries to run the node (Download compiled binaries from [Armada Alliance GitHub](https://github.com/armada-alliance/cardano-node-binaries)) 
2. gLiveView - Monitoring tool for the Cardano node
3. ScheduledBlocks - Tool to query the scheduled slots for a block production node. (Credits for this tool goes to [SNAKE POOL](https://github.com/asnakep/ScheduledBlocks))
4. Cardano Submit Transaction API - API to connect with a Cardano wallet (e.g. Nami) to send transactions via your own full node

```bash
cd ${HOME}/Cardano-node-docker/dockerfiles
sudo ./build.sh
```
The process might take some minutes.

Once the process is done, you can use the command to see the list of all Docker images:

```bash
docker images
```

You should see your Cardano node docker image in the list, e.g.

```bash
REPOSITORY              TAG            IMAGE ID       CREATED          SIZE
armada/armada-cn        1.34.1         da4414775ce6   37 seconds ago   619MB
<none>                  <none>         f3891eef21e4   3 minutes ago    1.09GB
```

All we need is the "armada/armada-cn" image. You can delete the others in the list to free up space on your harddrive, e.g.

```bash
docker rmi f3891eef21e4 
```

# 4. Start node

Let's first configure the run-node.sh script to match your host system environment.

```bash
cd ${HOME}/Cardano-node-docker/node
sudo nano run-node.sh
```

Edit the configuration section according to your setup.

* Note:If you are running the node as relay node, you can ignore the paramter CN_KEY_PATH.
* Important: Change the directory paths CN_CONFIG_PATH and CN_DB_PATH to the corresponding locations on your host. 

```bash
##Configuration for relay and block producing node
CNIMAGENAME="armada/armada-cn"                                   ## Name of the Cardano docker image
CNVERSION="1.35.0"                                               ## Version of the cardano-node. It must match with the version of the docker i>
CNNETWORK="testnet"                                              ## Use "mainnet" if connecting node to the mainnet
CNMODE="relay"                                                   ## Use "bp" if you configure the node as block production node
CNPORT="3001"                                                    ## Define the port of the node
CNPROMETHEUS_PORT="12799"                                        ## Define the port for the Prometheus metrics
CN_CONFIG_PATH="/home/julienterrier/Cardano-node-docker/node/files" ## Path to the folder where the Cardano config files are stored on the host>
CN_DB_PATH="/home/julienterrier/Cardano-node-docker/node/db"     ## Path to the folder where the Cardano database (blockchain) will be stored o>
CN_RTS_OPTS="+RTS -N2 -I0.1 -Iw3600 -A64m -AL128M -n4m -F1.1 -H3500M -O3500M -RTS"      ## RTS optimization parameters
CN_BF_ID=""                                                      ## Your blockfrost.io project ID (neededfor ScheduledBlock script)
CN_POOL_ID=""                                                    ## Your stake pool ID (needed for ScheduledBlock script)
CN_POOL_TICKER=""                                                ## Your pool ticker (needed for ScheduledBlock script)
CN_VRF_SKEY_PATH=""                                              ## Name of the vrf.skey file. It must be located in the directory CN_KEY_PATH
CN_KEY_PATH=""                                                   ## Path to the folder where the OP certificate and keys are stored on the host system
```

After making the changes, save and close the file.

`Ctrl+o & ENTER & Ctrl+x`

You can now run the docker image.

```bash
sudo ./run-node.sh
```

## Check the running status of the docker container

You can check the running status of the docker container at any time with:

```bash
docker ps -a
```

If the docker node started successfully, you might see something like this:

```bash
CONTAINER ID   IMAGE                     COMMAND                  CREATED          STATUS                    PORTS                                                                                      NAMES
fed0cfbf7d86   armada/armada-cn:1.34.1   "bash -c /home/carda…"   12 seconds ago   Up 10 seconds (healthy)   0.0.0.0:3001->3001/tcp, :::3001->3001/tcp, 0.0.0.0:12799->12798/tcp, :::12799->12798/tcp   cardano-node-testnet-1.34.1
```

You can also check the logs of the running cardano-node:

```bash
docker logs -f {CONTAINER ID}
``` 

To exit the logs press `Ctrl+c`

## Stop/Restart/Delete the Docker Cardano node

To stop the running Cardano node execute:

```bash
docker stop {CONTAINER ID}
```

A stopped container can be started again with:

```bash
docker start {CONTAINER ID}
```

A stopped container can also be deleted. Once deleted, it can not be started with the command above again.

```bash
docker rm {CONTAINER ID}
```

If you like to start the node again, after having removed the docker container, just run the run-node.sh script.

```bash
sudo ${HOME}/Cardano-node-docker/node/run-node.sh
```

## Monitor the Docker Cardano node with gLiveView

While the docker Cardano node is running, you can monitor its status with the tool gLiveView.

```bash
docker exec -it {CONTAINER ID} /home/cardano/pi-pool/scripts/gLiveView.sh
```

## Check the scheduled slots of the block production node

Our Docker image contains the ScheduledBlocks python script from [SNAKE pool](https://github.com/asnakep/ScheduledBlocks). This tool allows to
query the blockchain for the scheduled slots for your block production node.

Before using the script, make sure that the right configurations are set in our shell script run-node.sh. Set the following variables:

```bash
CN_BF_ID="mainnetd9PBzlK7KB7wWko8NTKUwJIsHfvEKNaV"               ## Your blockfrost.io project ID (for ScheduledBlock script)
CN_POOL_ID="c3e7025ebae638e994c149e5703e82619b31897c9e1d64fc684f81c2"   ## Your stake pool ID (for ScheduledBlock script)
CN_POOL_TICKER="MINI1"                                           ## Your pool ticker (for ScheduledBlock script)
CN_VRF_SKEY_PATH="scheduledblocks.vrf.skey"                      ## Name of the vrf.skey file. It must be located in the same directory as CN_KEY_PATH (for ScheduledBlock script)
CN_KEY_PATH="/home/julienterrier/Cardano-node-docker/node/files/.keys"  ## Path to the folder where the OP certificate and keys are stored on the host system
```

Start the ScheduledBlocks.py script and follow the instructions on the terminal:

```bash
docker exec -it {CONTAINER ID} python3 /home/cardano/pi-pool/scripts/ScheduledBlocks/ScheduledBlocks.py
```

# Run node in P2P (peer-to-peer) mode

{% hint style="warning" %}
Although P2P can be enabled on Node version 1.35.0, IOHK does not yet recommend using it because it has not yet been officially released.
{% endhint %}

In order for a node to connect to other peers in the network, a mechanism must be set in place. On Cardano the actual official mechanism
forsees the use of a static network topology file, where the IP adress and port number of known peers can be configured. To automate this process, a tool
called [TopologyUpdater](https://github.com/cardano-community/guild-operators/blob/alpha/docs/Scripts/topologyupdater.md) exists. IOHK is working on a more decentralized mechanism, called [peer-to-peer networking.](https://docs.cardano.org/explore-cardano/cardano-network/p2p-networking)
The P2P networking doesn't require the configuration of a static network topology file anymore. 

To configure P2P on a relay node, we need to make some changes in the *-topology.json and *-config.json files:

```bash
cd ${HOME}/Cardano-node-docker/node/files
sed -i 's+"TurnOnLogging": true,+"TurnOnLogging": true,\n  "TestEnableDevelopmentNetworkProtocols": true,\n  "EnableP2P": true,\n  "MaxConcurrencyBulkSync": 2,\n  "MaxConcurrencyDeadline": 4,\n  "TargetNumberOfRootPeers": 50,\n  "TargetNumberOfKnownPeers": 50,\n  "TargetNumberOfEstablishedPeers": 25,\n  "TargetNumberOfActivePeers": 10,+' *-config.json
```
 
Open the *-topology.json file with the nano editor and replace the entire content with:

```bash
sudo nano testnet-topology.json  ##use mainnet-topology.json for mainnet
```

Don't forget to enter the IP and Port of your block production node in the respective lines below:

```bash
{
  "LocalRoots": {
    "groups": [
      {
        "localRoots": {
          "accessPoints": [
            {
              "address": "[IP block Production node]",
              "port": [Port block production node]
            }
          ],
          "advertise": false
        },
        "valency": 1
      }
    ]
  },
  "PublicRoots": [
    {
      "publicRoots" : {
        "accessPoints": [
          {
            "address": "relays-new.cardano-mainnet.iohk.io",
            "port": 3001
          }
        ],
        "advertise": true
      },
      "valency": 1
    }
  ],
  "useLedgerAfterSlot": 0
} 
``` 

To configure P2P on the block production node, the steps are the same as above, only the content of the *-topology.json is different:

```bash
{
  "LocalRoots": {
    "groups": [
      {
        "localRoots": {
          "accessPoints": [
            {
              "address": "[IP Relay 1]",
              "port": [Port Relay 1]
            },
            {
              "address": "[IP Relay 2]",
              "port": [Port Relay 2]
            }
          ],
          "advertise": false
        },
        "valency": 2
      }
    ]
  },
  "PublicRoots": []
}
```

Your nodes are now ready to run in P2P mode.










