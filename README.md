During the previous months my interest in blockchain technologies has increased steadily. As somebody who has always been working in distributed systems, the advent of a technology which, beyond its obvious financial scope, promises to “decentralize everything”, sounds certainly appealing.

Even though we seem to be on a very early stage where new blockchains are still emerging to try to set the main foundations of the technology, Ethereum appeared so far to me as the most interesting, currently implemented blockchain, due to its flexible programmability via smart contracts.

# The problem

So I read a couple of books on Ethereum and Solidity and, as I usually do with any new technology, I wanted to quickly get my hands dirty and learn “by doing”. In this article I want to share my experience of trying to implement a simple task on Ethereum, and a few things I learned from it.

I wanted to implement a simple lottery-style game, one which simply involves buying a ticket and knowing immediately whether the ticket has a prize or not (kind of like “scratch & win” cards). 

This seems like a good use case for blockchain, as it involves instant rewarding. And it looks simple enough that it can be used as a learning task. But, is it really so simple?

# First thoughts

If you think about this problem from a traditional programming point of view, there isn’t really much to care about. One writes server-side business logic that checks whether a ticket has a prize or not, and secures this backend through a firewall. The burden is more in accepting the payment and rewarding the player.

However, if you want to implement this on a blockchain, you immediately get rid of all the financial burden, but you can’t really think about the business logic from such a “centralized” point of view.

For simplicity, we could reduce the rewarding logic to a simple random number. If we want to reward on average one out of every 10K virtual tickets, we could simply express our rewarding logic as:

```
  rand = random_number()
  reward = rand % 10000 == 0
```

I also thought about other possibilities like generating a true list of ticket IDs associated with a “reward” boolean attribute, but the implications of having such a dataset on the blockchain are too far fetched for a beginner’s task (could this dataset be saved in places like Swarm or IPFS? How would one go about invalidating already bought tickets? Could there be synchronization issues?).

So, my first stupid thought was: Isn’t there a random number generator built-in in Solidity? 

There isn’t. Ethereum was designed to be deterministic and transaction-replayable. Having a distributed ledger means reaching distributed consensus, and this is easily achieved if transaction outputs are deterministic. If there were such a random method, then smart contracts wouldn’t always be deterministic, every mining node could produce different results given the same data, and distributed consensus would hardly be reached.

# A naïve approach that doesn’t work

My second stupid thought was: Well, Ethereum addresses themselves could be seen as pseudo-random. Why not using them as random seed? So I coded my first smart contract, which assigns a reward based on the input address:

```javascript
pragma solidity ^0.4.11;

contract FirstIdea {

    mapping (address => uint) pendingWithdrawals;
    
    uint buyTicketWei;
    uint winningWei;
    uint winningOdds;

    function FirstIdea() {
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

I was happy to have completed my first smart contract and debugged it in Remix (a great tool!). But then, I realized, anyone could create a new Ethereum address from a random private key, and a simple brute force calculation would eventually produce a winning address, which could then be registered in the public blockchain and used to get a reward.

A small modification that could introduce a bit more of uncertainty would be accumulating hashes of all address that have played so far in the internal storage of the contract:

```
contract FirstIdeaModified {

    [...]

    bytes32 accumulator;

    function FirstIdea() {
	  [...]
      accumulator = keccak256(msg.sender);
    }
      
    function buyTicket() payable returns (bool) {
      accumulator = keccak256(accumulator, msg.sender);
      if (msg.value >= buyTicketWei) {
        [...]
        if(uint(accumulator) % winningOdds == 0) {
          // winning ticket
          [...]
    }
```

However, even if it looks safer, this still has the same problem: since smart contract data is public, anyone could look into the accumulated value “accumulator” at any given moment in time and perform the brute-force attack, in the hopes that there is enough time to call the contract with the new address before somebody else does it...
Using blockhashes

Then, my third stupid thought was: But there is actually pseudo-random-like data in the blockchain being continuously fed into smart contracts: the block hashes themselves, which are the output of SHA3 (Keccak256) functions, and which can’t be predicted. Can’t we just add them as random seeds?

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
                [...]
    }
```

Not so easily! The brute force attack is still a problem during the lifetime of the currently mined block. Anyone watching the blockchain could run it with the current blockhash as input. And miners would have a bigger advantage on this, since they are the ones finding the new hashes.

Looking into the future, however, is safe. This is the approach used by TrueFlip [ explain briefly here … ]. So it is actually ok to use, say, the N-th blockhash after the current one for a big enough N. The problem with this approach is, obviously, latency. For the use case I wanted to implement, one can’t really buy a ticket and wait for hours before knowing if it has a prize or not. 

And we still need to have a random decision on a per-ticket basis - if we only used the blockhash, we could only sell one ticket per block!

# Using an Oracle

But if randomness inside the blockchain seems to be a contradiction by itself, couldn’t we just feed it externally somehow? Enter the idea of “Oracles”.

There are actually a number of reasonable limitations due to how smart contracts and Ethereum were conceived. Notably, if the blockchain should be replayable and deterministic, it can’t rely on any external resource (kind of like purist unit testing philosophy). But nothing prevents a smart contract from accepting data “from the outside”!

The key idea here is: how can we make sure that the input received from an external oracle is legit? For example, if a smart contract aims to pay a certain amount depending on the weather conditions of a certain day, how could we validate that the input received from the outside is the actual, real weather data?

Fortunately, Oraclize offers TLS Notary, a technology that allows a smart contract to verify itself that the input received is the one produced by the external source that it asked for.

This is actually the approach used by iDice and its implementation can be seen on EtherScan.

In my case, and despite the robustness of this solution, I still wanted to find other solutions, mainly due to:

Using an external datasource like Oraclize makes it more difficult to test and debug the smart contract (however, they seem to offer a special version of Remix that solves this issue: https://docs.oraclize.it/#development-tools-encryption).
Throughput / latency / cost: How scalable is the contract given that it has to wait at least for the next block to be mined before making use of the fed output? Is it overkill to use such a service for the use case, from a cost point of view?

But mostly, I wanted to find other solutions just so that I could learn more!

# Signing

Inspired by the casino.dao white paper I found another idea: elliptic-curve cryptography as a source of randomness.

The underlying idea is that nobody can predict the output of an encrypted message, because only the private key owner knows it. A formalized algorithm with this idea is called “Signidice”.

I decided to implement an approximation of this idea for the use case:

```javascript
pragma solidity ^0.4.11;

contract Signidice {

    mapping (bytes32 => bool) playedTickets;
    mapping (bytes32 => bool) withdrawedTickets;
    
    uint buyTicketWei;
    uint winningWei;
    uint winningOdds;
    
    address contractKey;

    function Signidice() {
        buyTicketWei = 1000000000000000000;
        winningWei = buyTicketWei * 1;
        winningOdds = 1;
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

The advantage of this approach is that ticket issuing can be done completely off-chain, and we only use the blockchain when we want to withdraw a prize. The smart contract verifies that the played ticket Id has a prize as a function of a signed message from the smart contract creator. The contract verifies that the signed data is indeed signed by the smart contract creator by extracting the public key from the signed data’s elliptic curve. All thanks to the inherent cryptographic capabilities of Ethereum and Solidity.

Note how we need to append some constant data before verifying the signature, this is due to how signing works in Ethereum.

Not yet implemented, but necessary, would be the possibility of withdrawing the cost of the ticket after a certain amount of time. This is because we’re relying on the contract creator to answer with a signed message off-chain before we can proceed to check if the ticket has a prize or not.

However, relying on a single private key to finalize a transaction seems like a bottleneck and a security problem. And the transaction itself doesn’t really happen on the blockchain, but depends on a single process that signs data offline.

There is one more obvious issue here. The contract creator has the incentive to not answer and not sign the ticket if it knows that it will have a prize. The player can still be refunded the cost of the ticket, but the big prize would effectively be neglected. How could one go about this? The compensation for not having had a ticket signed should be as big as a prized ticket in order to eliminate this vicious incentive. Certainly a big motivation for the contract creator to have an always-up-and-running backend that signs as fast as possible!

(Note that it makes no sense for the smart contract creator to play against him/herself, as it would only win money that he/she already has).

# A federated solution?

I tried to dig a bit deeper and imagine what could be a solution for the last couple of issues found in the signing approach. On the one hand, everything should look a bit more decentralized. On the other hand, there should ideally be no incentive to cheat.

We could build an ecosystem on its own that would have several separate entities:

* Ticket issuers: They perform the signing process.
* Ticket buyers: They can buy tickets to any ticket issuer.
* Master contract: It “regulates” the ecosystem, rewards buyers and issuers.

[... work in progres ...]


