# Ideas on implementing a simple "scratch & win" game on the Blockchain

(*Disclaimer I: This article's blockchain technical level is mostly beginner and the code snippets shown here, although tested, are not meant to be used in production. The aim of this article is to share knowledge, discoveries and ideas. Any addition / comment would be more than welcome.*).

(*Disclaimer II: The topic of this article is mostly randomness on Ethereum, which has already ben discussed in several places like in [this thread](https://ethereum.stackexchange.com/questions/191/how-can-i-securely-generate-a-random-number-in-my-smart-contract). The value of this article lies more in the fact that it is structured around trying to get a practical idea implemented, the process I went through and the order in which I found out certain things.*)

During the previous months my interest in blockchain technologies has increased steadily. As somebody who has always been working in distributed systems, the advent of a technology which, beyond its obvious financial scope, promises to “decentralize everything”, sounds certainly appealing.

Even though we seem to be on a very early stage where new blockchains are still emerging to try to set the main foundations of the technology (see for example: [Tezos](https://www.tezos.com/), [Hashgraph](https://hashgraph.com/), [Universa](https://www.universa.io/)) Ethereum appeared so far to me as the most interesting, currently implemented blockchain, due to its flexible programmability via smart contracts.

# The problem

So I read a couple of books on Ethereum and Solidity and, as I usually do with any new technology, I wanted to quickly get my hands dirty and learn “by doing”. In this article I want to share my experience of trying to implement a simple task on Ethereum, and a few things I learned from it.

I wanted to implement a simple lottery-style game, one which simply involves buying a ticket and knowing immediately whether the ticket has a prize or not (kind of like “scratch & win” cards). 

![scratch_and_win](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a9/Scratch_game.jpg/1200px-Scratch_game.jpg)

This seems like a good use case for blockchain, as it involves instant rewarding. And it looks simple enough that it can be used as a learning task. But, **is it really so simple**?

# First thoughts

If you think about this problem from a traditional programming point of view, there isn’t really much to care about. One writes server-side business logic that checks whether a ticket has a prize or not, and secures this backend through a firewall. The burden is more in accepting the payment and rewarding the player.

However, if you want to implement this on a blockchain, you immediately get rid of all the financial burden, but you can’t really think about the business logic from such a “centralized” point of view.

For simplicity, we could reduce the rewarding logic to a simple random number. If we want to reward on average one out of every 10K virtual tickets, we could simply express our rewarding logic as:

```
  rand = random_number()
  reward = rand % 10000 == 0
```

I also thought about other possibilities like generating a true list of ticket IDs associated with a “reward” boolean attribute, but the implications of having such a dataset on the blockchain are too far fetched for a beginner’s task (could this dataset be saved in places like [Swarm](http://swarm-gateways.net/bzz:/theswarm.eth/)? How would one go about invalidating already bought tickets? Could there be synchronization issues?).

So, my first stupid thought was: Isn’t there a random number generator built-in in Solidity? 

There isn’t. Ethereum was designed to be deterministic and transaction-replayable. Having a distributed ledger means reaching distributed consensus, and this is easily achieved if transaction outputs are deterministic. If there were such a random method, then every mining node could produce different results given the same data, and distributed consensus would hardly be reached.

# A naïve approach that doesn’t work

My second stupid thought was: Well, Ethereum addresses themselves could be seen as pseudo-random (because they are taken from the output of Keccak256 hashes, check out this [great article](https://kobl.one/blog/create-full-ethereum-keypair-and-address/) if you're curious about it). Why not using them as random seed? So I coded my first smart contract, which assigns a reward based on the input address:

```javascript
contract FirstIdea {

    mapping (address => uint) pendingWithdrawals;
    
    uint buyTicketWei;
    uint winningWei;
    uint winningOdds;

    function FirstIdea() {
        // These numbers are stupid, just to make testing easier
        buyTicketWei = 1000000000000000000;
        winningWei = buyTicketWei * 1;
        winningOdds = 1;
    }
      
    function buyTicket() payable returns (bool) {
         if (msg.value >= buyTicketWei) {
            // if bigger, thanks!
            // (maybe withdraw-refund in later versions)

            if(uint(msg.sender) % winningOdds == 0) {
                // winning ticket
                pendingWithdrawals[msg.sender] = winningWei;
                return true;
            }
         } else {
            // invalid price paid
            // (withdraw-refund in later versions)
         }
         return false;
    }
    
    function withdrawPrize() returns (bool) {
        uint amount = pendingWithdrawals[msg.sender];
        if(amount > 0) {
            // Remember to zero the pending refund before
            // sending to prevent re-entrancy attacks
            pendingWithdrawals[msg.sender] = 0;
            msg.sender.transfer(amount);
            return true;
        }
        return false;
    }
}
```

I was happy to have completed my first (very dummy) smart contract and debugged it in [Remix](http://remix.ethereum.org/) (a great tool, by the way! Check out its debugger). But then, I realized, anyone could create a new Ethereum address from a random private key, and a simple brute force calculation would eventually produce a winning address, which could then be registered in the public blockchain and used to get a reward.

A small modification that could introduce a bit more of uncertainty would be accumulating hashes of all address that have played so far in the internal storage of the contract:

```javascript
contract FirstIdeaModified {

    // [...]

    bytes32 accumulator;

    function FirstIdea() {
      // [...]
      accumulator = keccak256(msg.sender);
    }
      
    function buyTicket() payable returns (bool) {
      accumulator = keccak256(accumulator, msg.sender);
      if (msg.value >= buyTicketWei) {
        // [...]
        if(uint(accumulator) % winningOdds == 0) {
          // winning ticket
          // [...]
    }
```

However, even if it looks safer, this still has the same problem: since **smart contract data is public**, anyone could look into the accumulated value “accumulator” at any given moment in time and perform the brute-force attack, in the hopes that there is enough time to call the contract with the new address before somebody else does it...

# Using blockhashes

Then, my third stupid thought was: There is actually pseudo-random-like data in the blockchain being continuously fed into smart contracts: the **blockhashes** themselves, which are the output of SHA3 (Keccak256) functions, and which can’t be predicted. Can’t we just add them as random seeds?

```javascript
function buyTicket() payable returns (bool) {
  if (msg.value >= buyTicketWei) {
     // if bigger, thanks!
     // (maybe withdraw-refund in later versions)
         
     var blockNumber = block.number;
     var blockHashNow = block.blockhash(blockNumber);

     if(uint(keccak256(msg.sender, blockHashNow)) % winningOdds == 0) {
         // winning ticket
         pendingWithdrawals[msg.sender] = winningWei;
         // [...]
     }
```

Not so easily! The brute force attack is still a problem during the lifetime of the currently mined block. Anyone watching the blockchain could run it with the current blockhash as input. And miners would have a bigger advantage on this, since they are the ones finding the new hashes.

Looking into the future, however, is safe. This is the approach used by [TrueFlip](https://trueflip.io/) (they run a lottery based on the next hash after the closing time of the lottery). So it is actually ok to use, say, the N-th blockhash after the current one for a big enough N. The problem with this approach is, obviously, **latency**. For the use case I wanted to implement, one can’t really buy a ticket and wait for hours before knowing if it has a prize or not. And we still need to have a random decision on a per-ticket basis - if we only used the blockhash, **we could only sell one ticket per block**!

# Using an Oracle

But if randomness inside the blockchain seems to be a contradiction by itself, couldn’t we just feed it externally somehow? Enter the idea of “Oracles”.

There are actually a number of reasonable limitations due to how smart contracts and Ethereum were conceived. Notably, if the blockchain should be replayable and deterministic, it can’t rely on any external resource (kind of like purist unit testing philosophy). But nothing prevents a smart contract from accepting data “from the outside”!

The key idea here is: how can we make sure that the input received from an external oracle is legit? For example, if a smart contract aims to pay a certain amount depending on the weather conditions of a certain day, how could we validate that the input received from the outside is the actual, real weather data?

Fortunately, [Oraclize](http://www.oraclize.it/) offers [TLS Notary](https://tlsnotary.org/), a technology that allows a smart contract to verify itself that the input received is the one produced by the external source that it asked for.

This is actually the approach used by [iDice](https://idice.io/) and its implementation can be seen [on EtherScan](https://etherscan.io/address/0xe6517b766E6Ee07f91b517435ed855926bCb1AaE#code).

In my case, and despite the robustness of this solution, I still wanted to find other solutions, mainly due to:

* Using an external datasource like Oraclize makes it more difficult to test and debug the smart contract (however, they seem to offer a special [version of Remix](https://docs.oraclize.it/#development-tools-encryption) that solves this issue).
* Throughput / latency / cost: How scalable is the contract given that it has to wait at least for the next block to be mined before making use of the fed output?

But mostly, I wanted to find other solutions just so that I could learn more!

# Signing

Inspired by the [Casino.DAO White Paper](https://github.com/DaoCasino/Whitepaper/blob/master/DAO.Casino%20WP.md) I found another idea: elliptic-curve cryptography as a source of randomness.

The underlying idea is that nobody can predict the output of an encrypted message, because only the private key owner knows it. A formalized algorithm with this idea is called “[Signidice](https://github.com/DaoCasino/Whitepaper/blob/master/DAO.Casino%20WP.md#35-algorithm-implemented-in-mvp-of-daocasino-protocol)”.

(There is also two-phase commit/reveal like [RANDAO](https://github.com/randao/randao/blob/master/README.md) but I assume this approach isn't really suitable for one-to-one games).

I decided to implement an approximation of this idea for the use case:

```javascript
contract Signidice {

    mapping (bytes32 => bool) playedTickets;
    mapping (bytes32 => bool) withdrawedTickets;
    
    uint buyTicketWei;
    uint winningWei;
    uint winningOdds;
    
    address contractKey;

    function Signidice() {
        // These numbers are stupid, just to make testing easier
        buyTicketWei = 1000000000000000000;
        winningWei = buyTicketWei * 1;
        winningOdds = 1;
        // I'd change this to a static address in Remix to make testing easier
        contractKey = msg.sender;
    }
      
    function buyTicket(uint ticketNumber) payable returns (bool) {
        if (msg.value >= buyTicketWei) {
            // if bigger, thanks!
            // (maybe withdraw-refund in later versions)
           
            // calculate "ticketId"
            bytes32 ticketId = sha3(msg.sender, ticketNumber);
            
            if(playedTickets[ticketId]) {
                // already played, refuse
                // (withdraw-refund in later versions)
                return false;
            }
            
            playedTickets[ticketId] = true;
            return true;
        } else {
            // invalid price paid
            // (withdraw-refund in later versions)
        }
        return false;
    }
    
    // v, r, and s need to be calculated externally!
    // I use web3.eth.sign for that
    function withdrawPrize(uint ticketNumber, uint8 v, bytes32 r, bytes32 s) returns (bool) {
        // calculate "ticketId"
        bytes32 ticketId = sha3(msg.sender, ticketNumber);
        
        if(withdrawedTickets[ticketId]) {
            // already withdrawed, refuse
            return false;
        }
            
        bytes memory prefix = "\x19Ethereum Signed Message:\n32";
        bytes32 prefixedHash = sha3(prefix, ticketId);
        address pubKey = ecrecover(prefixedHash, v, r, s);

        if(pubKey == contractKey) {
            // valid signed data, ticket can be validated
            if(uint(r) % winningOdds == 0) {
                // winning ticket, pay
                withdrawedTickets[ticketId] = true;
                msg.sender.transfer(winningWei);
                return true;
            }
        }
        
        return false;
    }
}
```

It's actually not trivial to sign and do "erecover", check out [1](https://ethereum.stackexchange.com/questions/15364/totally-baffled-by-ecrecover) and [2](https://github.com/ethereum/web3.js/issues/392) if you're interested.

The smart contract verifies that the played ticket id has a prize as a function of a signed message from the smart contract creator. The verifying method has to be called by the same ticket buyer, so the signing process is supposed to happen off-chain in this first version (this isn't really optimal since the contract doesn't implement any escrow yet, we'll improve this later). The contract verifies that the signed data is indeed signed by the smart contract creator by extracting the public key from the signed data’s elliptic curve.

However, relying on a single private key like a bottleneck and a security problem. And the transaction itself doesn’t really happen on the blockchain, but depends on a single process that signs data offline.

There is one more obvious issue here. The contract creator has the incentive to not sign the ticket if it knows that it will have a prize. How could one go about this? We could implement escrow/refund for this. But the compensation for not having had a ticket signed should be as big as a prized ticket in order to eliminate this vicious incentive. Certainly a big motivation for the contract creator to have an always-up-and-running backend that signs as fast as possible!

(Note that it makes no sense for the smart contract creator to play against him/herself, as it would only win money that he/she already has).

# A federated solution?

I tried to dig a bit deeper and imagine what could be a solution for the last couple of issues found in the signing approach. On the one hand, everything should look a bit more decentralized. On the other hand, there should ideally be no incentive to cheat.

We could build an ecosystem on its own that would have several separate entities:

* **Ticket issuers**: They perform the signing process.
* **Ticket buyers**: They can buy tickets to any ticket issuer.
* **Master contract**: It “regulates” the ecosystem.

The key ideas being:

* Issuers create a contract through the master contract, issue tickets and reward prizes.
* Deposits are spread out among ticket issuers. So if one ticket issuer gets hacked, only a smart portion of the money is affected.
* Issuers work with their own money so they have no incentive to play their own tickets.
* Issuers could cheat and not sign winning tickets to avoid paying a prize, but this behavior could be registered by the master contract.

Let's see how the first implementation of this idea looks like:

```javascript
contract Master {
    address[] issuerContracts;

    mapping (address => uint) issuerTimeouts;
    mapping (address => uint) issuerSoldTickets;
    mapping (address => uint) issuerRewardedTickets;
    
    uint createIssuerWei = 1000000000000000000;
    
    function createContract (bytes32 name) payable {
        if(msg.value >= createIssuerWei) {
            address newContract = new Issuer(name, msg.sender, this);
            // Transfer the diposit directly to the issuer contract
            newContract.transfer(msg.value);
            issuerContracts.push(newContract);
        }
    }
    
    function issuerTimedOut(Issuer issuer) {
        issuerTimeouts[msg.sender] += 1;
    }
    
    function issuerSoldTicket(Issuer issuer) {
        issuerSoldTickets[msg.sender] += 1;
    }
    
    function issuerRewardedTicket(Issuer issuer) {
        issuerRewardedTickets[msg.sender] += 1;
    }
}

contract Issuer {

    mapping (bytes32 => bool) playedTickets;
    mapping (bytes32 => bool) withdrawedTickets;
    
    mapping (address => uint) escrows;
    mapping (address => uint) playingTime;
    
    uint buyTicketWei;
    uint winningWei;
    uint winningOdds;
    
    address contractKey;
    bytes32 _name;
    
    uint issuerTimeout;

    Master _masterContract;
    
    function Issuer(bytes32 name, address owner, Master masterContract) {
        _name = name;
        _masterContract = masterContract;
        // Stupid values, just to make testing easier
        buyTicketWei = 1000000000000000000;
        winningWei = buyTicketWei * 1;
        winningOdds = 1;
        issuerTimeout = 1;
        contractKey = owner;
    }
      
    function buyTicket(uint ticketNumber) payable returns (bool) {
        if (msg.value >= buyTicketWei) {
            // if bigger, thanks!
            // (maybe withdraw-refund in later versions)
            
            // calculate "ticketId"
            bytes32 ticketId = sha3(msg.sender, ticketNumber);
            
            if(playedTickets[ticketId]) {
                // already played, refuse
                // (withdraw-refund in later versions)
                return false;
            }

            // In case the issuer times out and we need to refund
            escrows[msg.sender] = buyTicketWei;
            playingTime[msg.sender] = block.timestamp;           
            playedTickets[ticketId] = true;
            _masterContract.issuerSoldTicket(this);
            return true;
        } else {
            // invalid price paid
            // (withdraw-refund in later versions)
        }
        return false;
    }
    
    function withdrawEscrow() returns (bool) {
        // all uninitialised state has zero values
        if(escrows[msg.sender] > 0 && playingTime[msg.sender] > 0) {
            if(block.timestamp - playingTime[msg.sender] > issuerTimeout) {
                // ticket won't be played, refund
                var refund = escrows[msg.sender];
                escrows[msg.sender] = 0;
                playingTime[msg.sender] = 0;
                _masterContract.issuerTimedOut(this);
                msg.sender.transfer(refund);
                return true;
            }
        }
        return false;
    }
    
    function validateTicket(uint ticketNumber, address player, uint8 v, bytes32 r, bytes32 s) returns (bool) {
        if(msg.sender == contractKey) {
            // calculate "ticketId"
            bytes32 ticketId = sha3(player, ticketNumber);
            
            if(withdrawedTickets[ticketId]) {
                // already withdrawed, refuse                
                return false;
            }
                
            bytes memory prefix = "\x19Ethereum Signed Message:\n32";
            bytes32 prefixedHash = sha3(prefix, ticketId);
            address pubKey = ecrecover(prefixedHash, v, r, s);
    
            if(pubKey == contractKey) {
                // make sure there is no escrow left after calling this method
                escrows[player] = 0;
                playingTime[player] = 0;

                // valid signed data, ticket can be validated
                if(uint(r) % winningOdds == 0) {
                    // winning ticket, pay
                    withdrawedTickets[ticketId] = true;
                    _masterContract.issuerRewardedTicket(this);
                    player.transfer(winningWei);
                    return true;
                }
            }
        
            return false;
        }
    }
    
    function releaseFunds(uint amount) {
        if(msg.sender == contractKey) {
            msg.sender.transfer(amount);
        }
    }
}
```

The master contract can be called to obtain a new contract that can be used as ticket issuer (**contracts can create other contracts**, how cool is that?). The issuer contract is bound to an external Ethereum address that can always be used for releasing some funds. The `validateTicket()` method is now to be called by the issuer, and if a certain time passes (measured as time difference **between block times**, which is the way to measure time inside Ethereum), the player could always ask for a refund via `withdrawEscrow()`.

The issuer contracts communicate with the master contract internally, so the master contract knows how issuers are doing: how many tickets they sold, how many prizes they gave and how many timeouts they had. Right now nothing is done with this information, but obviously this could be used in a variety of ways (for reputation, issuer blacklisting, etc).

Certainly less simple than one could have imagined, and we have just started. We should check all possible edge cases, make sure the contract can be destroyed and its funds released via `suicide()` method, test the contract properly on a real blockchain (like the [Ropsten](https://ropsten.etherscan.io/)), and so on.
