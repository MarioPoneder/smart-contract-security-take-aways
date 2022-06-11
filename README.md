# My key take-aways from reviewing vulnerable Solidity smart contracts
The list contains general pitfalls when developing smart contracts in Solidity as well as vulnerabilites which arise due to the interaction of multiple (DeFi) contracts.  
For more in-depth explanations, see also my documented solutions for the smart contract hacking challenges:
[Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/index.html) <[solutions](https://github.com/MarioPoneder/damn-vulnerable-defi-solutions/blob/master/SOLUTIONS.md)>
and [Ethernaut](https://ethernaut.openzeppelin.com/) <[solutions](https://github.com/MarioPoneder/ethernaut-solutions/blob/master/SOLUTIONS.md)>.

1.  The contract constructor is `function contractName(args)` up to Solidity 0.4.21 and `constructor(args)` in all newer versions.
2.  Do not rely on pseudorandomness based on values like `blockhash(blockNumber)` or `block.timestamp`.
    Both the timestamp and the block hash can be influenced by miners to some degree.
3.  Do not use `tx.origin` for authorization since it makes a contract susceptible to phishing attacks.
4.  Use the `SafeMath` library for arithmetic operations in contracts with Solidity versions < 0.8.0 to avoid "silent" integer over-/underflows.
5.  Do not allow `delegatecall`s in a contract with a target function or even target contract address which can be freely specified by the caller.
    This allows the caller do anything in the context (`msg` context and storage) of your contract.
    For example, a token's `approve` function could be called on behalf of your contract.
6.  Keep in mind that even a contract without a payable fallback function can be forcefully funded with some ETH using the `selfdestruct` function.
7.  Never store secret data (e.g. a password) in a contract since its whole storage can be read by anyone, even private variables.
8.  Always check return values (especially the success of `call` and `delegatecall`) and decide if the calling contract should revert on failure.
    Beware of denial-of-service (DoS) attacks where a callback to a user contract intentionally fails (`revert`, `assert`) and therefore blocks the execution of your contract.
    Keep in mind, that even just sending ETH to a contract invokes the fallback function which could potentially revert on purpose.
9.  Set a gas limit when making `call`s to user contracts. Otherwise 63/64 of the remaining gas is forwarded to the call.
    If all of that gas is used, there might not be enough left to finish the execution of your contract. (possible DoS attack, even if the call succeeds)
10. Protect contract functions against reentrancy when they call functions of a user contract in order to eliminate vulnerabilites of this category.
    Make sure that the reentrancy protection cannot be circumvented through another unprotected contract function.
11. Checking for the assembly command `extcodesize(caller())` to return 0 does not assure that `msg.sender` is an EOA, because it will also return 0
    when your contract is called from a potential attacker contract's constructor. (Contract code is stored after constructor execution.)
12. Keep in mind that if your contract is derived from a base contract, all `public/external` functions of the base contract are still callable if not
    explicitly overridden and protected by a modifier in your contract.
13. Whenever it is necessary to use `delegatecall`, make sure that there are no storage collisions with the called (library/implementation) contract.
    Always review the storage layout of the involved contracts.
14. The address of a new contract is generated in a deterministic way based on the creator's address and transaction nonce.
15. Below solidity version 0.5.0, it was possible to compile contracts with uninitialized local storage variables which can point to unexpected storage locations.
16. A low-liquidity trading pair on a DEX, which computes the swap price based on the liquidity pool balances, can be easily manipulated with a flash loan to get cheap tokens.
17. Keep in mind that `msg.value` is also preserved through `delegatecall`s and calls to non-`external` functions of the same contract.
    This way, the value might be counted multiple times in a poorly designed contract although the sender paid just once.
18. Make sure that initializable implementation contracts which are designed to be called through a proxy contract are also initialized in their own context.
    Otherwise (depending on the implementation) anyone can initialize and therefore take ownership of the contract.
19. Be careful with `require` and `assert` statements because they might unexpectedly break your contract and therefore lock the deposited funds forever.
    For example, do not rely on `address(this).balance` or `token.balanceOf(address(this))` in assertions because your contract can be forcefully funded with ETH or tokens via
    `selfdestruct(...)` or `token.transfer(...)` (from another contract) which can lead to unexpected balances that might break your assertions.
20. Protect functions that accept e.g. a flash floan and return an additional fee from being called by anyone. Otherwise someone could potentially drain your contract by
    repeatedly calling the function and collecting the fee.
21. When offering flash loans, make sure that the receiver cannot deposit the loan in the pool (counted as depositor) while still satisfying the payback condition.
22. Token distributions (rewards) based on user-triggered snapshots can be manipulated using flash loans to get more tokens (rewards).
    The same is true for governance actions.
23. When your contract relies on oracles to access off-chain data (e.g. token prices), make to sure to use multiple oracles which cannot be manipulated by a single (centralized) entity.
24. Although a transaction is always reverted when a `require` statements fails, make sure to perform the necessary checks before executing a governance action or transferring funds.
    Otherwise, a malicious executed action or fallback function (when transferring funds) could make a subsequent `require` statement succeed which would have failed when checked in advance. 
