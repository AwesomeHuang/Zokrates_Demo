# Zokrates_Demo
A demo using Zokrates to build an encrypted transaction on Ethereum/Qtum based on zkSNARKs algorithm.

> This is a proof-of-concept implementation. It has not been tested for production.

## Zokrates Introduction

[ZoKrates](https://github.com/Zokrates/ZoKrates) is a toolbox for zkSNARKs on Ethereum. It helps you use verifiable computation in your DApp, from the specification of your program in a high level language to generating proofs of computation to verifying those proofs in Solidity.

Using Zokrates, we can implement the zkSNARKs in the platform which supports Solidity.

The goal of zero-knowledge proofs is for a *verifier* to be able to convince herself that a *prover* possesses knowledge of a secret parameter, called a *witness*, satisfying some relation, without revealing the witness to the verifier or anyone else.

In order to let platform like Ethereum/Qutm support zkSNARKs, we use Zokrates to generate the proof.

## Zokrates Installation

Zokrates official manual can be found in [Here](https://zokrates.github.io/gettingstarted.html).

### One-line install
We provide a one-line install for Linux, MacOS and FreeBSD:

```bash
curl -LSfs get.zokrat.es | sh
```

### Docker
ZoKrates is available on Dockerhub.

```bash
docker run -ti zokrates/zokrates /bin/bash
```


From there on, you can use the ```zokrates``` CLI.

### From source
You can build the container yourself from [source](https://github.com/ZoKrates/ZoKrates/) with the following commands:

```bash
git clone https://github.com/ZoKrates/ZoKrates
cd ZoKrates
cargo +nightly build --release
cd target/release
```

## Implementation

### Design

In this chapter we aim to give an overview of [zkSNARKs](https://en.wikipedia.org/wiki/Non-interactive_zero-knowledge_proof) from a practical viewpoint. We will treat the actual math as a black box but will try to develop some intuitions around how we can use them.

In this Demo, we want to encrypt the balance to protect user's privacy. Usually, user's information like balance was storing in clear text, which will expose user's privacy. In this demo, we want to replace it with hashvalue. 

Here is a simple example of how zkSNARKs can help with privacy on Ethereum/Qtum.

Suppose we have a simple token contract. Normally a token contract would have at its core a mapping from addresses to balances:

```
mapping (address => uint256) balances;
```

For example, Alice has 100 coins and Bob has 50 coins, Alice wants to send 10 coins to Bobs using a transaction. If all the information are stored by clear text, we can verify the balance easily. But if both the balance and the transfer value are stored using hash value, we can hardly verify the correctness because no one can knows the value behind the hash. So we must use zero-knowledge proof to verify the correctness.

We are going to retain the same basic core, except replace a balance with the hash of a balance:

```
mapping (address => bytes32) balanceHashes;
```

We are not going to hide the sender or receiver of transactions, but we’ll be able to hide the balances and sent amounts. 

**Remember: Two zk-SNARKs will be used to send tokens from one account to another, one proof created by the sender and one by the receiver.**

Thus what our zkSNARKs would need to prove is that this holds as well as that the updated hashes matches the updated balances.

### Program

The main idea is that the sender will use their starting balance and the transaction value as private inputs, and hashes of starting balance, ending balance and value as public inputs. Similarly the receiver will use starting balance and value as secret inputs and hashes of starting balance, ending balance and value as public inputs.

Below is the program we will use for the sender zkSNARKs, where ***private field*** represents private input and ***field*** represent public input:

```
import "hashes/sha256/512bitPacked.code" as sha256packed

def main(private field value, private field before, field valueHash, field beforeHash, field afterHash) -> (field):
	priBefore = sha256packed([0, 0, 0, before])
	priAfter = sha256packed([0, 0, 0, before-value])
    field result = if(\
    	value > before &&\
    	priBefore[0] == beforeHash &&\
    	priAfter[0] == afterHash \
    ) then 1 else 0 fi
    return result

```

The program used by the receiver is below:

```
import "hashes/sha256/512bitPacked.code" as sha256packed

def main(private field value, private field before, field valueHash, field beforeHash, field afterHash) -> (field):
	priBefore = sha256packed([0, 0, 0, before])
	priAfter = sha256packed([0, 0, 0, before+value])
    field result = if(\
    	priBefore[1] == beforeHash &&\
    	priAfter[1] == afterHash \
    ) then 1 else 0 fi
    return result

```

The programs check that the sending balance is larger than the value being sent, as well as checking that all hashes match. The most important difference between sender and receiver is that sender need to check the balance is larger than the value but receiver is no need.

### Compile and Generate Proof

The sender and receiver are compiled in the same way. Take sender as an example:

> Make sure you have finished installation Zokrates, and enter the environment.

Create a file ***sender.code*** to store the code.

```bash
mkdir sender
cd sender
vim sender.code #Paste the code for sender in this file
```

Then, compile the code and run the setup:

```bash
# compile
zokrates compile -i sender.code
# perform the setup phase
zokrates setup
```

After that, we can get a circuit used to execute a proof, in this program, we have 5 inputs, so we must input 5 variables. 

For example, sender has 1000 coins, he wants to transfer to receiver 50 coins, after that sender still has 950 coins, so in this example, we must input 50, 1000, sha256(50), sha256(1000), sha256(950) :

```bash
# execute the program
zokrates compute-witness -a 50 1000 242738482787324818092317501628658271637 853498718274837825789312739748392789743 438758372489912996993285694393204086976
```

Then, we can get a witness file, we use this file to generate a proof:

```bash
# generate a proof of computation
zokrates generate-proof
```

Finally, we can get a json file which contains the proof, verified person can use this to proof:

```json
{
        "proof": {
            "a": ["0x25eaee22d216321bfdb2106e4d82cb57cb68ff5b2bc2392f3af46136c32864a9", "0x1ae67dfbcc13ff83a3101107a22c01003026d497ec0e43412cfdcd3f71a9bff2"],
            "b": [["0x1e7c39359a8f77f3ef922d14d99421df80c1e482b84e440f0ffd7c04ba774ecb", "0x13905c2bc7e57fb43005f1418c8ed2fef5901199a2ddef3a10c83d408d241f08"], ["0x2d54fb29e5dca94e115f5e5638e1d0b76a82d3b2eaaaa65558947c9a4154bda6", "0x1b1822bf4fc6de9eadde6bea6cceb55584a20fa1601be543b2b93d85463ea2c8"]],
            "c": ["0x05f3c493eacb68b4349e5321fe45af6eca70dc3a7abd31b29c1902a69c3aa0ef", "0x09fe5c2631a47feea9c30d8f60348dcc86e5b3e18c980275528cdefc2454c8cf"]
        },
        "inputs": ["0x00000000000000000000000000000000b69dbb3437a3e859225943db8ef8c595", "0x000000000000000000000000000000028219dfb80d5c593e8837d46577ece6ef", "0x000000000000000000000000000000014a15c9ee54bfb820a51232819ea418c0", "0x0000000000000000000000000000000000000000000000000000000000000001"]
    }
```

### Solidity Contract Generate

Using Zokrates, we can generate a verifier, which can be deployed in the Ethereum/Qtum.

```bash
zokrates export-verifier
```

After that, users can find a file named ***verifier.sol*** in the folder. Then we use the [Remix](https://remix.ethereum.org/) to test the verifier contract.

#### Paste the code into the Remix

<img src="./pic/pic1.png" alt="pic1" style="zoom:50%;" />

#### Compile it

<img src="./pic/pic2.png" alt="pic2" style="zoom:50%;" />

#### Find Verifier Class and Delopy

<img src="./pic/pic3.png" alt="pic3" style="zoom:50%;" />

#### Use VerifyTx to Execute Proof

<img src="./pic/pic4.png" alt="pic4" style="zoom:50%;" />

#### Input the Parameters in Json and Verify

If the program execute successfully and return true, means the user is honest and has pass the proof, if user is dishonest, the program won't pass.

<img src="./pic/pic5.png" alt="pic5" style="zoom:50%;" />

## About Qtum

[Qtum](https://qtum.org/en) is compatible with the Bitcoin and Ethereum ecosystems and aims at producing a variation of Bitcoin with Ethereum Virtual Machine (EVM) compatibility. 

Note that differently to Ethereum, the Qtum EVM is constantly backwards compatible. Pursuing a pragmatic design approach, Qtum employs industry use cases with a strategy comprising mobile devices. 

The latter allows Qtum promoting blockchain technology to a wide array of Internet users and thereby, decentralizing PoS transaction validation.

<img src="./pic/pic6.jpeg" alt="pic6" style="zoom:50%;" />
