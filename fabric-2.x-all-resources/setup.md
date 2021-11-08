# Hyperledger Fabric 2.3+ Setup 

Make sure you are using the `bash` shell. If you are using `fish` or `zsh`, just enter the following command to enter the bash shell. 

```bash 
bash
``` 

## Setting up the Environment 

Let's first go ahead and setup docker and some prerequisites. 

```bash 
sudo apt install git curl jq 
sudo apt-get -y install docker-compose
docker --version 
sudo systemctl start docker
sudo systemctl enable docker 
echo $USER    # ensure your username shows here 
sudo usermod -a -G docker $USER 
``` 

Make sure you log out and log back in to ensure correct permissions are set for your user. Otherwise, docker gives errors. 

```bash 
docker ps 
``` 

If that works correctly i.e. does not throw any errors, you may proceed. Otherwise, first setup the permissions through `usermod` above. 


Let's now setup node. The latest supported version currently is 12. Future versions should be usable and will be mentioned on the Fabric site. 

```bash 
curl -fsSL https://deb.nodesource.com/setup_12.x | sudo -E bash - 
sudo apt update 
sudo apt-get install -y nodejs  
```

Ensure that correct versions of Node and NPM are installed: 

```bash 
npm  --version    # shouled be 6.x 
node --version    # should be 12.x 
```

## Downloading and Installing Fabric Binaries 

Setting up Fabric is now extremely straight-forward using the procided `test-net` scripts. Let's first create a fabric directory to ensure everything remains in one place. 

```bash 
mkdir ~/fabric 
cd fabric 
``` 

Let's download the Fabric binaries in this folder. 


```bash 
curl -sSL https://bit.ly/2ysbOFE | bash -s -- 2.3.1 
```

This will clone the repo and setup docker images using Docker Compose. I strongly recommend you read through all the outputs. 

## Setup the Test Network 

First, create a channel on which our chaincode (or smart contract) will be deployed: 

```bash 
cd fabric-samples/test-network 
./network.sh up createChannel -c channel1 -ca  
``` 

Verify that new containers are indeed up. 

```bash 
docker ps 
``` 

Now let's create a package for our chaincode. First, set up the dependencies and then create a package. 

```bash 
cd ../asset-transfer-basic/chaincode-javascript 
npm install  
``` 

Make sure everythingn installed correctly and without any errors. 

```bash 
cd ../../test-network 
export FABRIC_CFG_PATH=$PWD/../config/ 
echo $FABRIC_CFG_PATH 
```

Now, we must make sure that fabric commands are in the PATH. The following commands adds it to the current session but you should add it to your `~/.bashrc` file as well for future use. 

```
peer   # ensure it's there. If not, set the path 
export PATH=$PATH:/home/nam/fabric/fabric-samples/bin/ 
peer   # should work now 
```

Now, let's create a pacakge from our chaincode source. 


```bash {cmd=true}
peer lifecycle chaincode package basic.tar.gz \
      --path ../asset-transfer-basic/chaincode-javascript \
      --lang node \
      --label basic_1.0
```

Let's set the environment variables needed through the provided script. 

```bash 
source ./scripts/envVar.sh 
``` 

## Installing the Chaincode to Channel 

```bash 
setGlobals 1 
```

This selects the first organization. Now we can go ahead and install the chaincode on the channel. First let's save the certificate file locations for both organizations and the orderer. 

```bash 
# the $PWD is necessary as the commands need absolute paths, relatives don't work 

export ORDERER_CERTFILE=$PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

export ORG1_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

export ORG2_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
```

First, install the package. 

```bash 
peer lifecycle chaincode install basic.tar.gz 
``` 

Then, do the same for Organization 2. 

```bash 
setGlobals 2
peer lifecycle chaincode install basic.tar.gz 

setGlobals 1  # back to org1 
```

Take note of the Package ID given back. We need this to refer to the chaincode in later commands. 

```bash 
peer lifecycle chaincode queryinstalled \
      --peerAddresses localhost:7051 \
      --tlsRootCertFiles $ORDERER_CERTFILE 
```

Using the output of the above command, set the package ID env variable. (It's ok if the queryinstalled one gives you an error. We can proceed without it.)

```bash 
# make sure you use the actual package ID and not copy/paste the one below 
 
export PKGID=basic_1.0-94c84bb2cc404bab2d945d62dc8d8d3837e2074966495d037b27ecfdf7fe171a
```


Once we have that, we need to approve the chaincode definition for organizations. 

```bash 
peer lifecycle chaincode approveformyorg \
      -o localhost:7050 \
      --ordererTLSHostnameOverride  orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE  \
      --channelID channel1 \
      --name basic \
      --version 1 \
      --package-id $PKGID \
      --sequence 1  
```

And do the same for the second organization: 

```bash 
setGlobals 2 

peer lifecycle chaincode approveformyorg \
      -o localhost:7050 \
      --ordererTLSHostnameOverride  orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE  \
      --channelID channel1 \
      --name basic \
      --version 1 \
      --package-id $PKGID \
      --sequence 1  
```

Let's commit the chnages made to the chaincode. We have to specify peers here explicitly. When we later use the chaincode through our code, we will not have this issue. 

```bash 
peer lifecycle chaincode commit \
      -o localhost:7050 \
      --ordererTLSHostnameOverride orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE \
      --channelID channel1 --name basic \
      --peerAddresses localhost:7051 --tlsRootCertFiles $ORG1_PEER_CERTFILE \
      --peerAddresses localhost:9051 --tlsRootCertFiles $ORG2_PEER_CERTFILE \
      --version 1 --sequence 1  
```

And confirm that it went through. 

```bash 
peer lifecycle chaincode querycommitted \
      --channelID channel1 --name basic \
      --cafile  $ORDERER_CERTFILE
```

Finally, see `docker ps` to see chaincode containers running. 
```bash 
docker ps 
```

## Working with the Chaincode through CLI 

Let's first initialize our ledger. For this, we will call the `InitLedger` function that we defined in `chaincode-javascript` project. This can be done using the `invoke` subcommand. 

```bash 
peer chaincode invoke \
      -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE  \
      -C channel1 -n basic \
      --peerAddresses localhost:7051 --tlsRootCertFiles $ORG1_PEER_CERTFILE \
      --peerAddresses localhost:9051 --tlsRootCertFiles $ORG2_PEER_CERTFILE \
      -c '{"function": "InitLedger", "Args": []}'
```

The last line is the most important one and is pretty straight-forward. If we just want to read the contents, it's much simpler. 

```bash 
peer chaincode query -C channel1 -n basic -c '{"Args":["GetAllAssets"]}' | jq 
```

And we can also query a particular asset using the `ReadAsset` function, passing it the target asset name. 

```bash 
peer chaincode query -C channel1 -n basic -c '{"Args":["ReadAsset", "asset6"]}' | jq 
```


While you're doing this, you can open the docker logs in another console to see how stuff is going. 

```bash 
# Watch the past logs 
docker-compose -f docker/docker-compose-test-net.yaml logs  | more 

# or put it on follow as a running log 
docker-compose -f docker/docker-compose-test-net.yaml logs -f 
```

Of course, this doesn''t work in another console as we don't have the proper environment variables set. Let's see what needs to be done for that. 

```bash 
bash   # make sure it's bash 

# head over to the proper working folder 
cd ~/fabric/fabric-samples/test-network 

# set the environment variables 
export PATH=$PATH:~/fabric/fabric-samples/bin/
export FABRIC_CFG_PATH=$PWD/../config/

export ORDERER_CERTFILE=$PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

export ORG1_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

export ORG2_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt


source ./scripts/envVar.sh
# Select which organization you're emulating 
setGlobals  1   
```


## How to Clean Everything  
Sometimes you might run into really problematic issues or just want to experiment with the setup from scratch. Since everything is running on top of docker, this is really easy. Run the following to remove EVERYTHING and start from scratch.

```bash 
# See what stuff is avaialble 
docker images -a 
docker volume ls 

# Remove ALL the docker containers (Not just for fabric, ALL!!)
docker rm -vf $(docker ps -a -q) 

# .. and the images 
docker rmi -f $(docker images -a -q) 

# Volumes are not deleted by default 
docker volume ls 

# ... but we can get rid of those too 
docker system prune -a --volumes 

# Verify that everything is indeed gone 
docker volume ls 
docker images 
docker container ls 

# And just for final measure 
docker system prune --all 


# finally, get rid of the files as well. 
# Make sure you copy your code before doing this (if any)   
cd ~/fabric 
sudo rm -rf fabric-samples
```


# Accessing the Chaincode through Code 

Of course, we don't want to use the CLI to do everything. We need to be able to do this through Node code. 

We will create a simple project for this based on the example provided in `fabric-samples/asset-transfer-basic/application-javascript`. However, we want to clean the code a bit so that it's easier to work with. 

Let's start by creating a node project in our `fabric` folder. 

```bash 
cd ~/fabric 
mdkir asset-app 
cd asset-app 

# initialize a node project 
npm init 
``` 

Install the required packages in this project. 

```bash 
npm install fabric-ca-client 
npm install fabric-network 
``` 

## Helper Functions 

We need to first create some helper function. First is a function to get the organization's configurations. Complete sources of these files are available in the resources. In this file, we wil only discuss 

```javascript 
exports.buildCCPOrg1 = function() { 
    const ccpPath = path.resolve(__dirname, '..', 'test-network', 
    'organizations', 'peerOrganizations', 'org1.example.com', 
    'connection-org1.json'); 

    const fileExists = fs.existsSync(ccpPath); 
    if(!fileExists) { 
        throw new Error("Cannot find: ${ccpPath}"); 
    }

    const contents = fs.readFileSync(ccpPath, 'utf8'); 

    const ccp = JSON.parse(contents); 

    console.log("Loaded network config from: ${ccpPath}"); 

    return ccp; 
}
``` 

We then create a function for creating a wallet. This wallet will hold the keys and other crypto material used througout the code. The waller can store in-memory (for testing) or in a folder (which is what we will use). 

```javascript 
exports.buildWallet = async function (Wallets, walletPath) { 
    let wallet; 

    if (walletPath) { 
        wallet = await Wallets.newFileSystemWallet(walletPath);     
        console.log("Built wallet from ${walletPath}"); 
    } else { 
        wallet = await Wallets.newInMemoryWallet(); 
        console.log("Build in-memory wallet"); 
    }

    return wallet; 
}
``` 

There's also a function to pretty-print javascript. This is straight-forward. 

```javascript 
exports.prettyJSONString = function(inputString) { 
    return JSON.stringify(JSON.parse(inputString), null, 2);
}
```


## Performing Administrative Actions 

There are some functions that will set up the admins and users. Logic for this is in the `caActions.js`Let's take a look at those here. 

First function is to load the credentials used for performing actions. 

```javascript 
function buildCAClient(FabricCAServices, ccp, caHostName) { 
    const caInfo = ccp.certificateAuthorities(caHostName); 
    const caTLSCACerts = caInfo.tlsCACerts.pem; 
    const caClient = new FabricCAServices(caInfo.url, {
        trustedRoots: caTLSCACerts, verify: false 
    }, caInfo.caName); 

    console.log("Built a CA client named: ${caInfo.caName}"); 
    return caClient; 
};
``` 

This function will be called with the following information: 

```javascript 
let ccp = helper.buildCCPOrg1(); 

const caClient = buildCAClient(
                    FabricCAServices, 
                    ccp, 
                    'ca.org1.example.com'
                ); 
```

We also need to create an administrative account. This will be a onetime task. We first create the enrollment information along with its X.509 certificate. Then we put it in the wallet. 

```javascript 
async function enrollAdmin(caClient, wallet, orgMspId) { 
    // ... 
        
        const enrollment = await caClient.enroll({ 
            enrollmentID: adminUserId, 
            enrollmentSecret: adminUserPasswd
        }); 

        const x509Identity = { 
            credentials: {
                certificate: enrollment.certificate, 
                privateKey: enrollment.key.toBytes()
            }, 
            mspId: orgMspId, 
            type: 'X.509'
        }; 

        await wallet.put(adminUserId, x509Identity);
    // ... 
};
```

Creating an everyday user is similar but it's done more often. We load the admin credentials, use those to create the user and save the user's credentials in the wallet. 

```javascript 
async function registerAndEnrollUser(caClient, wallet, orgMspId, userId, affiliation){ 
    // ... 

    // Must use an admin to register a new user
    const adminIdentity = await wallet.get(adminUserId);

    // ... 

    // build a user object for authenticating with the CA
    const provider = wallet.getProviderRegistry().
                        getProvider(adminIdentity.type);
    const adminUser = await provider.getUserContext(adminIdentity, adminUserId);


    const secret = await caClient.register({
                          affiliation: affiliation,
                          enrollmentID: userId,
                          role: 'client'
    }, adminUser);

    const enrollment = await caClient.enroll({
      enrollmentID: userId,
      enrollmentSecret: secret
    });

    const x509Identity = {
      credentials: {
        certificate: enrollment.certificate,
        privateKey: enrollment.key.toBytes(),
      },
      mspId: orgMspId,
      type: 'X.509',
    };

    await wallet.put(userId, x509Identity);

    // ...
};
```

There are two more function `getAdmin` and `getUser` which are basically interfaces for the above two functions. 

Finally, we have some helper code so that we can call these functions from the command line. 

```javascript 
let args = process.argv; 

if (args[2] === 'admin') { 
    getAdmin(); 
} else if (args[2] === 'user') { 
    let org1UserId = args[3]; 
    getUser(org1UserId); 
} else { 
    console.log("Invalid command");
}
```


## Performing Actions on the Ledger Itself 

Once we have the users and admins, we can perform the actual chaincode logic. Logic for this is in the `ledgerActions.js`. 

Relevant fragments of code are discussed here. First part is the connection information. Think of this as being similar to creating a database connection. 

```javascript 
const ccp = helper.buildCCPOrg1();
const wallet = await helper.buildWallet(Wallets, walletPath);
const gateway = new Gateway();

await gateway.connect(ccp, {
            wallet,
            identity: org1UserId,
            discovery: { enabled: true, asLocalhost: true }
        });

```

Then we establish the connection to the actual chaincode. Again, think of this as selecting a specific database from our connection. 

```javascript 
const network = await gateway.getNetwork(channelName);  // channel1 
const contract = network.getContract(chaincodeName);    // basic
```


Using the actual chaincode functions is no extremely easy. Simply call the relevant function like so: 

```javascript 
await contract.submitTransaction('InitLedger'); 
``` 

This function essentially calls the function in the chaincode. This can be seen as the `InitLedger` function in `fabric-samples/asset-transfer-basic/chaincode-javascript/lib/assetTransfer.js`. 

The only difference is that since we are using `subimtTransaction`, this will go to the orderer and the whole voting and consensus algorithm will work and the transaction will be added to the blockchain. From a programming perspective, all of this is hidden behind this one simple function call. 

(Note that this line has actually been commented out in the file since we have already initialized the ledger previously in our workflow here.)

Very similarly, we have the `GetAllAssets` and `ReadAsset` functions. 

```javascript 
let result = await contract.evaluateTransaction('GetAllAssets'); 
// ... 
let result = await contract.evaluateTransaction('ReadAsset', asset); 
// ...
let result = await contract.submitTransaction('CreateAsset', 
                                'asset101', 
                                'violet', 
                                '5', 
                                'Snnorre 3', 
                                '1300'); 
```

The file also has some code fragments that allow passing comments into the code. So, we can call it using the following commands. 

```bash 
node ledgerActions.js GetAllAssets 
node ledgerActions.js ReadAsset asset1 
node ledgerActions.js CreateAsset 
```

Of course, you can add node code for exposing these calls through a REST API and call those REST APIs from your front-end such as React, Vue etc. 