One thing the Blockchain is very often connected to is the immutability of data. For long time it was "Once it's deployed, it cannot be altered". That is still true for historical transaction information. But it is not true for Smart Contract storage and addresses. 

###  The main Reason for Upgrades

- Fixing bugs, adding features.
- Decentralized governance.

One of the main problems with Solidity is that the storage for normal Smart Contracts is bound to the Smart Contract Address. If you deploy a new version you also start with an empty storage.

**Overview of Standards for Smart Contract Upgrades**

- The Eternal Storage Pattern. Actually it was initially proposed by Elena Dimitrova on her Blog.

- We expand with Proxies where it all started (apparently):
the upgradeable.sol gist from Nick Johnson, Lead developer of ENS & Ethereum Foundation alum.

- EIP-897: ERC DelegateProxy
Created 2018-02-21 by Jorge Izquierdo and Manuel Araoz

- EIP-1822: Universal Upgradeable Proxy Standard (UUPS)
Created 2019-03-04 by Gabriel Barros and Patrick Gallagher

- EIP-1967: Standard Proxy Storage Slots
Created 2019-04-24 by Santiago Palladino That's OpenZeppelin is using.

- EIP-1538: Transparent Contract Standard Created 2018-10-31 by Nick Mudge

- EIP-2535: Diamond Standard Created 2020-02-22 by Nick Mudge

- Not really a standard, but I think Metamorphic Smart Contracts should be covered as well. Those are Smart Contracts that get re-deployed to the same address with different logic using EIP-1014 CREATE2. It's said to be wild magic in Ethereum.

## Eternal Storage without Proxy

Loss of data during re-deployment ---> Solution: separate logic from storage.

In the Eternal Storage pattern, we move the storage with setters and getters to a separate Smart Contract and let only read/write the logic Smart Contract from it.

```
//SPDX-License-Identifier: MIT

pragma solidity 0.8.1;


 contract EternalStorage{

    mapping(bytes32 => uint) UIntStorage;

    function getUIntValue(bytes32 record) public view returns (uint){
        return UIntStorage[record];
    }

    function setUIntValue(bytes32 record, uint value) public
    {
        UIntStorage[record] = value;
    }


    mapping(bytes32 => bool) BooleanStorage;

    function getBooleanValue(bytes32 record) public view returns (bool){
        return BooleanStorage[record];
    }

    function setBooleanValue(bytes32 record, bool value) public
    {
        BooleanStorage[record] = value;
    }


}

library ballotLib {

    function getNumberOfVotes(address _eternalStorage) public view returns (uint256)  {
        return EternalStorage(_eternalStorage).getUIntValue(keccak256('votes'));
    }

    function setVoteCount(address _eternalStorage, uint _voteCount) public {
        EternalStorage(_eternalStorage).setUIntValue(keccak256('votes'), _voteCount);
    }
}

contract Ballot {
    using ballotLib for address;
    address eternalStorage;

    constructor(address _eternalStorage) {
        eternalStorage = _eternalStorage;
    }

    function getNumberOfVotes() public view returns(uint) {
        return eternalStorage.getNumberOfVotes();
    }

    function vote() public {
        eternalStorage.setVoteCount(eternalStorage.getNumberOfVotes() + 1);
    }
}
```

This is a simple voting Smart Contract. You call vote() and increase a number - pretty basic business logic. Under the hood is the magic.

- First we need to deploy the Eternal Storage. This contract remains a constant and isn't changed at all. 
- Then we deploy the Ballot Smart Contract, which will take the library and the Ballot Contract to do the actual logic. 

Under the hood, a library does a *delegatecall*, which executes the libraries code in the context of the Ballot Smart Contract. If you were to use msg.sender in the library, then it has the same value as in the Ballot Smart Contract itself. 

Let's say we found a bug, because everyone can vote as many times as they want. We fix it and re-deploy only the Ballot Smart Contract (neglecting that the old version still runs and that there is no way to stop it without extra code).

**RESULT**

- Only the Library changed. The Storage is exactly the same as before. But how to deploy the update?

- Re-Deploy the "Ballot" Smart Contract and give it the address of the Storage Contract. That's all. 

- Address of Contracts change - this can also be good for transparency reasons. E.g. you run an online service and fees change for new signups.

**Advantages**

- Relatively easy to understand: It doesn't involve any assembly magic at all. If you come from traditional software development, these patterns should look fairly familiar.

- Would also work without Libraries, just a Storage Smart Contract running under its own address.

- Eliminates the Storage Migration after Contract Updates.

- Easier to audit, easier to grasp, and less error prone than any other proxy pattern. 


**Disadvantages**

- Quite difficult access pattern for variables.

- Doesn't work out of the box for existing Smart Contracts like Tokens etc.


