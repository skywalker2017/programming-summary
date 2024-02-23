# solidity-summary
solidity grammer, vulnerabilities, EIPs

# summarize

## proxy contract

### transparent proxy pattern

+ how to proceed if the logic contract has a function with the same signature or the 4-byte identifiers are same.
+ whether to call logic function or proxy management function is distinguished by the message sender.
    * if the caller is the admin of the proxy, the proxy will not delegate any calls.
    * if the caller is any other address, the proxy will always call the logic contract no matter if it matches one of
      the proxy's contract
+ the owner of the proxy contract should be moved to a dedicated address. If using zeppelinOS to build upgradeable
  contract, this will be managed by an app contract, which is created whenever you publish your project to network.

### UUPSUpgradeable

+ upgrading logic should be included in implementation contract by importing UUPSUpgradeable.sol
+ we can modify the authority of upgrading by override _authorizeUpgrade function
+ the address of logic contract is stored in the specific slot of proxy contract storage space.

### DEX-Uninswap

#### V1

##### overview

+ each erc20 token has its own contract, and can be created using factory contract by anyone
+ liquid provider need to deposit equivalent value of eth and relevant erc20 token
+ pool token is sent to providers to represent their contribution, minted when deposit and burned when withdraw
+ directly swap between erc20-erc20 in a single transaction is also available.
+ a liquid provider fee 0.3% is taken out of each trade, and added to reserves. it is a payout to liquid providers when
  they retrieve their portion of reserve, and increase the amount of fee revenue generated.
+ only one exchange can be registered to the factory, to encourage concentration of reserve.
+ custom liquid pool can be used in one side of erc20-erc20 swap. only if the contract follows uninswap's interface.

##### sdks

+ Liquidity:
    + signature: exchange.methods.addLiquidity(min_liquidity, max_tokens, deadline).send({ value: ethAmount })
    + the first deposit of the reserve of the token set the initial exchange rate between erc20 and eth;
    + since ethAmount is fixed, the amount of erc20 token that deposit can be fluctuated between when a transaction is
      signed and when it actually executed. max_tokens is to bound the exchange rate
    + liquidity token represent the relative contribution a provider made to the reserve, it can also fluctuate between
      the time gap. min_liquidity is to constrain the affection of such fluctuation.
+ Trade:
    + following this formula if eliminate provider fee rate: outputAmount = inputAmount*(outputReserve/(
      inputReserve+inputAmount))

#### V3

##### overview

+ there can be multiple pools for an asset pair, distinguished by fee rate
+ concentrated liquidity: liquidity can be allocated with in a custom price range instead of previous version which is
  from 0 - infinity, it could lead to a more efficient usage of token's liquidity especially for stablecoin such as DAI.

#### Difference Between V3 and V4
+ in v4 liquid provider can introduce private logic into their liquid pool like before or after swap, or before or after liquid pool position is changed.

#### contract

## maker DAO

### concept

#### maker vault

+ borrow dai through locking crypto asset as collateral
+ repay dai and fee to retrieve collateral

#### liquidation

+ a vault is liquidated if the collateral value falls too low
+ part of collateral is auctioned to cover the outstanding debt plus penalty fee
+ the covered dai is then burned to maintain the total supply
+ vault owner than receive the leftover collateral

#### system line of defence

+ supply and demand of vault(and thus dai) is influenced by stability fee, dai saving rate, debt ceiling adjustment
+ liquidation when vault value get closer to debt, a liquidation of that vault would be triggered
+ MKR mint/burn
+ emergency shutdown

#### system stabilizer

+ flopper: when debt reaches limitation bidders compete with decreasing amount of mkr for fixed amount of debt. After
  auction finished, dai was burned to cancel the debt, and mkr is minted to the winner
+ flapper: when the surplus dai reaches limitation, bidders compete with increasing amount of mkr for fixed amount of
  surplus dai, when auction finished, surplus dai was sent to winner and mkr is burned

## solidity

### tips

#### libraries embed-library and linked-library

+ embed library seems like the library code pasted on your code, it has only internal function and don't need to be
  deployed on the blockchain. but it cost more when using these libraries for it enlarge the size of the contract.
+ linked library has public or external function and need to be deployed on the blockchain.

#### private internal public external

+ function marked as private can only be called from inside the contract
+ function marked as internal can be called both from inside and inherit contract
+ function marked as public can be accessed both from internal and external
+ function marked as external can only be accessed from other contract

#### difference between external and public

+ public can be called both internally and externally
+ public arguments are copied to memory, which is much more expensive than read from calldata directly
+ external can only be called from outside the contract
+ external arguments are read directly from calldata, which is cheaper than memory location
+ It's much cheaper to use external if the function is only called from outside the contract

#### maximum size of a contract

48 kilobytes after shanghai upgrade, 24 kilobytes before that

#### how to avoid a contract exceed size limit

+ splitting contract
+ summarize common used functions into libraries and avoid duplicating in contract
+ using inherit
+ using external contract
+ using interface

#### differences between create and create2

they are all for creating contract in solidity, while in create the address of a contract is determined by creator's
address and nonce. In create2 address is determined by creator's address salt and hash of init code, create2 is widely
used in proxy contract and erc4337 standard

#### how to calculate a contract address created by create
+ keccak(rlp.encode([sender, nonce]))
+ for eoa: nonce0= address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), _origin, bytes1(0x80))))));
+ for contract: nonce1= address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), _origin, bytes1(0x01))))));
+ 0xd6 represents the total length of the list is 22 bytes (0xd6-0xc0) address(20), 0x94(1), nonce(1)
+ 0x94 represents the length of _origin is 20 (0x80+0x14)

#### difference between rlp encode and abi encode

+ abi: used to interact with ethereum ecosystem, both from outside the blockchain and contract-to-contract, its not self-describing and need schema to decode;
+ rlp: encode sheme used in ethereum, used for encodeing transaction, block, its self-describing.


#### major arithmetic changes in solidity v0.8.0

safemath library is introduced into this new version which lead to execution revert when overflow or underflow occurred,
but we can use uncheck modifier to bypass(circumvent) such check,if we are convinced of overflow or underflow won't
happen

#### difference between require and assert

they all need a conditional statement as one of their parameters, if the statement returns false require syntax will
revert the execution while assert syntax will just return false and let the program keep running

#### what is shanghai upgrading

shanghai upgrading allows those validator withdraw their stacked eth locked in the network

#### how to avoid infinite loop

+ caution when calling unknown external contract
+ avoid recursive callings

#### how to send eth to a smart contract without fallback, payable and receive

+ use other contract selfdestruct to send eth to the target contract
+ set the receiver of block award to the target contract
+ set the receiver address as the target contract when withdraw the stacked eth in beacon chain

#### difference between tx.origin and msg.sender

tx.origin means original signer of a transaction, while msg.sender means the directly caller of a contract. tx.origin
can never be a contract. to avoid phishing attack, it's recommended that tx.origin should never be used.

#### difference between memory and calldata in solidity

+ memory and calldata are both modifiers that describe location of a variable. memory is used to store temporary variables
during execution in a function, while calldata represent the arguments passed in from external contract, and it is
non-modifiable
+ It's much conciser if you want to copy an array in storeed calldata by using slice syntax, for it's only copy the reference of the parameter
+ If you want to copy an array in memory, you have to create a new arry in memory and copy the element one by one
+ however for example if you want to convert an array of address to an array of IERC20 in memmory you can simpley use assembly equal to convert it.


#### stack, memory, calldata, storage
+ stack: EVM opcode pop information from or push data onto the stack
+ calldata: the data field of a transaction located in memory, read-only.
+ memory: information store accessible for the duration of a transaction.
+ storage: persistent data store
+ code: executing code and static data storage.
+ logs: write-only logger.

#### what is abi and what does it work for

abi is some kind of interface of a contract, it generated by compiler and defines how to interact with the contract on
blockchain. It can also be used for encode or decode the data that send or receive from the transaction receipt

#### what is fallback function in solidity

it should be marked as external and runs when there is no other functions that match the function identifier or when the
contract receive ethers without any data, fallback function must be marked as payable if doing so.

#### what does receive function work for

receive function is called when a contract receive plain ethers without data, it should be marked as external and
payable

#### difference between pure and view

view modifier is used when state is read-only for a function, which means it won't change state when calling the
function, it only retrieves data from contract. While pure is used when the function is used to platform computations
without access to the state of the contract.

#### what is modifier in solidity

+ modifier is some kind of extra code added to a function, mainly used to verify the parameter, restrict access, guard
against reentrancy attack
+ you can use underscore to replace the the original function body
+ you can use modifier to add extra code to a function, but you can't use modifier to add extra code to a contract


#### how to declare a modifier in solidity

declaring a modifier is similar to declaring a function, but only marked as modifier instead, underscore symbol can be
used in modifier, ~~representing the position of the original function code in the modifier~~ representing where the
function body should be inserted in the definition of modifier

#### differences between struct and array

+ struct can not be returned if the function marked as external

#### tips when using mapping in solidity

+ key type of mapping should be built-in type such as string, bytes or any contract or enum type, reference types are
  not allowed
+ key data is not stored in mapping, so it doesn't have length and cannot be cleared. If it is necessary to clear a
  mapping, iterable mapping should be used
+ mapping can only occur in data location of storage thus can be state variable. Mapping cannot be used as parameters or
  return value of public visible function
+ if you mark a mapping storage variable as public, solidity creates getter for this mapping,you can pass in a key and
  get the value in return

#### differences between delegate call, call and static call

+ they are all low-level functions in solidity that allow a contract execute code from another contract which return
  success condition (bool) and data (memory bytes) which can be decoded by using abi
+ delegate call typically used when calling a contract in which library codes are stored. Using present contract's
  context and storage, the called contract only offer its code. (proxy contract)
+ call on the other hand, the called contract use its own state and context, it means msg.sender would be the address of
  caller contract.(normally call another contract)
+ staiccall is similar to call, but will revert if state changed during execution (simulation call in erc4337)

#### how to represent fix point arithmetic in solidity

+ represented as fixedM*N M means total bits for this number while N means bits reserved for decimal part

#### what is ERC165 used for

+ to check if a function is supported by a contract, pass in the identifier of a function and return true or false
+ can be used to check whether a contract can receive ethers before sending

#### What is a slippage parameter useful for

+ there is a time interval between sending time and executing time of a transaction, the exchanging rate between two
  token can be fluctuated violently, slippage parameter is used to alleviate such influence
+ it can also be used by defending sandwich attack.

#### What does ERC721A do to reduce mint costs? What is the tradeoff?

+ a batch of nft token can be pre-minted by the owner in erc721a

#### difference between hard fork and soft fork

+ literally there are two chain after a hard fork, new block generated can not be verified by the old version of
  validator, like the hard fork which lead to the birth of etc and beacon chain of erc2.0 get online.
+ a soft fork mean the old version of validator is still functional, which mean two version of node can work
  simultaneously in a single network

#### Checks-Effects-Interactions

A best practice for writing secure smart contract. It is used to prevent reentrancy attack.

+ Checks: This involves implementing input validations to ensure that arguments passed are valid and the function is ready to be executed.
+ Effects: All state modifications should happen before any external call is made. This step involves optimistically modifying the state variables to a valid state in the protocol.
+ Interactions: Any external call should be the very last thing done in a function.

#### Difference between transfer, send and call

+ it send forward 2300 gas when using transfer and it got revert if failed, there are two conditions that might got revert
  + gas not enough
  + the receiving contract reject the payment (not implement receive() or fallback())
+ send() will return false if the payment is reject, it also send forward 2300 gas same with transfer()
+ call will send all the left gas forward, and return a bool and a bytes, the bool represent whether the call is success and the bytes represent the return or revert message.
  ```
  (bool success, bytes memory data) = _addr.call{value: msg.value, gas: 5000}(
            abi.encodeWithSignature("foo(string,uint256)", "call foo", 123)
        );
  ```

#### gas cost when using sload

after eip2929, the first time load from a slot will cost 2100 gas, the subsequent load will cost 100 gas, for the use of cache when load slot from stateDB

#### why construct function shouldn't be used in upgradeable contract

because the construct function cannot be delegate called, and cannot change the state stored in the proxy contract

#### what will happen if you call a 0 address or a self-destructed address

if you call it by using delegatecall or call, it will return false. if you call it normally, it will revert

#### difference between erc777 and erc20

the most efficient feature of erc777 is hook, which means the receiver can react to receiving tokens

#### difference between openzeppelin safeMint and mint

in safeMint if the target is a smart contract, it must implement `onERC721Received` function. This function is designed to prevent issues such as tokens being stuck in contracts that do not support erc721 transfer.

#### why float-point arithmetic doesn't supported in solidity

float-point arithmetic cost large amount of resources, instead solidity use fixed-point, **fixed** is represented by fixed32x18, 32 is the integer part while 18 is decimal part. On the other hand one can use `FixedPoint` library that offer common arithmetic operations for fixed-point number with 32bit.

#### delegatecall and CODESIZE

if you delegate call a implement contract and the contract use CODESIZE opcode, the opcode would return the code size of the caller contract

#### why we sign the hash of a message but not the message it self?

here is the code of using erecover:
```solidity
function verifySignature(bytes32 messageHashHash, uint8 v, bytes32 r, bytes32 s) public pure returns (address) {
        // Verify the signature and recover the signing address
        address signer = ecrecover(messageHashHash, v, r, s);
        return signer;
    }
```

It may lead to vunerablility if we sign the message directly. when calling erecover, we use the hash of the thing that we signed. Attacker could manufact hash clash to use our signature.

#### selfdestruct

it remove the bytecode and storage of a contract, send the remaining eth to a specific address, and refund the leftover gas.

selfdestruct(reciever) undermines a contract’s immutability.

if a self destructed contract has some erc20 tokens, they can never be retrived.

selfdestruct has been deprecated since Shanghai upgraged EIP-6049, here are alternatives of selfdestruct: pauseable contract, proxy conract, withdraw pattern

#### why openzeppelin Proxy.sol overwrite the free memory point

alloc new memory slot cost more gas, so Proxy use yul to reuse the slot.
here is the code of fallback function:
```solidity
fallback() external payable {
        address _impl = _implementation;
        assembly {
            // get the free memory point
            let ptr := mload(0x40)
            // copy the calldate that used for delegate call to that point
            calldatacopy(ptr, 0, calldatasize())
            let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)
            let size := returndatasize()
            // copy the return data to free memory point
            returndatacopy(ptr, 0, size)
            // return or revert the original message
            switch result
            case 0 { revert(ptr, size) }
            default { return(ptr, size) }
        }
    }
```

#### how to retrieve the revert error message

reverted error message is encoded in abi, the first 4-byte indicate the error selector, the following 32 bytes indicate the start offset the error message represented in bytes. The next 32 bytes is the length of the msg, so you can use the following assembly code to retrieve the error message
```
bytes memory revertReason;
 assembly {
    revertReason := add(_msg, 68)
}
```
or you can revert the raw bytes in assembly, it's much cheaper. 32 means skip the fisrt byte32 which represent the length
```solidity
(bool success, bytes memory data) = address(mustRevert).call(abi.encodeWithSignature("mustRevert(uint256)", _case));
assembly {
   revert(
      // Start of revert data bytes. The 0x20 offset is always the same.
      add(data, 32),
      // Length of revert data.
      mload(data)
      )
   }
```
compared with the code in Proxy.sol, it doesn't add 32-bytes because the result of assembly call didn't experience abi encode.

#### RLP encoding rules

special length value: 
+ [0x00, 0x7f]: single byte;
+ [0x80, 0xb7]: string of 0-55 bytes long, 0x80 plus the length of the string
+ [0xb8, 0xbf]: string more than 55 bytes long, 0x80 plus the length of the length of the string, 255 maxium
+ [0xc0, 0xf7]: total payload of a list with 0-55 bytes long;
+ [0xf8, 0xff]: total payload of a list more than 55 bytes long; 255 maxium

#### the process of deplopying UUPS contract

```solidity
pragma solidity ^0.8.19;

import {UUPSUpgradeable} from "@openzeppelin/contracts/proxy/utils/UUPSUpgradeable.sol";
import {OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract TestUUPS is OwnableUpgradeable, UUPSUpgradeable {
    uint256 public value;

    function initialize(uint256 value_) public initializer {
        value = value_;
        __Ownable_init();
    }

    function _authorizeUpgrade(
        address newImplementation
    ) internal override onlyOwner {}
}


function testDeployAndInitialize() internal {
        address implementation = address(new TestUUPS());
        uint256 value = 42;

        bytes memory data = abi.encodeCall(TestUUPS.initialize, value);
        address proxy = address(new ERC1967Proxy(implementation, data));

        TestUUPS testUUPS = TestUUPS(proxy);
        assertEq(testUUPS.value(), value);
    }
```
1. deploy implementation contract
2. deploy proxy contract and delegatecall initialize function of impl contract

#### why upgradeToAndCall should be set onlyProxy in UUPS upgradeable pattern

normally, when you deploy a UUPS upgradeable contract you only call the initialize function through delegatecall when you create the proxy contract. Which means it only initialize the slot in proxy contract, and the owner of impl contract is still 0x0..., you can claim the ownership of impl contract by call it directilly, and then you can call the upgradeToAndCall and update the logic address slot of impl contract to a malicious address, normally it won't be a problem because the logic address of proxy contract doesn't change unless there is a selfdestruct during upgradeToAndCall, and it will delete the logic contract. So upgradeToAndCall is determined to be called only proxy. Or you can initialize the logic contract in the first time.

#### why UUPS is much more gas efficient than Transparency upgradeable pattern

// todo try it in forge

#### why can't dynamically change the length of an array store in memory

It's originate from the way the data stored in memory one after another, and there no space between and never deallocated.
the size of array in memory have to be preallocated like this
`bytes memory newBytes = new bytes(10);`

#### how to make sure that initialize() is called only once in openzeppelin

by checking `address(this).code.length == 0` to make sure the calling is issued from a constructor. This mechanism save a storage variable representing whether the contract is initialized. On the other hand onlyDelegate modifier is introduced to prevent the logic contract controlled by malicious account and got self-destructed.

### attacks

#### gas griefing

caller send exact gas for the vunerable contract to execute, but not enough for calling other contracts, which lead to the revert of calling. This kind of attack might cause unpredictable operation like infinate loop or DoS

#### timestamp dependencies

it is vulnerable to generate random number rely on block timestamp, because block timestamp can be detected and
manufactured. If someone has the source code of the contract together with the authority to package block at the same
time, he has the ability to decide what timestamp in block is. in this way he can manipulate the random number in the
contract

#### front-running attacks

there is a time gap between a user submit a transaction and the transaction be actually executed, malicious actor can
advantage from such mechanism by adding his own transaction with higher gas into the tx-pool. it makes the attacker's
transaction executed before the victim's. An example of front-running attack is sandwich attack. which means putting the
victim's transaction between two attacker's transactions, and manipulating the price of asset in decentralize exchange
to cause loss of victim.

here is the solutions

1. using gasThrottle to limit the usage of gas
2. giving a lower maximum slippage

#### reentrancy attack

a victim contract call malicious contract, while malicious has logic that callback to victim contract and lead to
recursive call.

solutions

1. call another contract after modify the state Checks-Effects-Interactions
2. be caution when calling unknown external contracts
3. use gasThrottle and limit gas usage
4. using reentrancy guard or mutex to forbid reentrancy execution.

#### read-only reentrancy attack

it may occur if a read-only function call a write function in a contract, attacker can call the read-only function repeatly to modify the state of the contract. The problem is fixed by the latest solidity compiler.

#### phishing attack

the malicious contract deceive the owner of vulnerable contract to act that only owner can perform, the attacker
contract can impersonate the owner and perform unauthorized actions on the vulnerable contract.

### gas consumption optimizing

#### use ++i instead i++
++i is pre-increment operator, it means that it use the value after it increments, and there is no additional operations
i++ is post-increment operator, which mean it is used first and than increase the value, and it cost some extra gas

#### use calldata instead of memory

#### use external instead of public when the function can only be used from outside
the parameters are copied from calldata to memory if you uese 

#### store states in specific slot like the use of slot0 of uniswap

#### using assembly to achieve type convertion


## ethereum

### tips

#### difference between btc and ethereum

+ utxo model: Unspent transaction outputs, they work like cash, not belonging to specific person and can be gathered and
  passed to someone in a transaction. If there are some exchanges that need to be paid back, utxos will split and some
  of them will be returned to the original sender as exchange.
+ balance model: like bank account stores the total balance of token your address has.
+ consensus: btc is pow, eth is pos at present
+ smart contract: ethereum and evm can execute specific code on blockchain, which is called smart contract, btc doesn't
  have such ability

#### procedure of verifying signature of eth transaction

1. retrieve signature from transaction r,s,v
2. calculating recovery id using v
3. using recovery id and r, s and hash of original transaction hash to recover public key
4. calculating keccak hash of the public key and taking the last 20 bytes of it as address
5. compare the calculated address with the original address in the transaction

#### what is dao

decentralized autonomous organization collectively-owned

#### consensus algorithm

1. pow: brutal calculating the hash of a block, if the result is less than certain value, the block is considered to be
   legal
2. pos: proof of stake, validators stake cryptocurrency to a pool to _secure the network_
3. dpos: similar to pos, stakeholders vote for delegates who are responsible for validating transactions and generate
   blocks

#### summarize pbft

+ an algorithm that make a set of node with one primary node and other backup nodes achieve consensus. it includes three
  phases. pre-prepare, prepare, commit.
+ pre-prepare: primary node send pre-prepare message to all backup nodes, backup nodes validate the message and enter
  prepare phase
+ prepare: backup nodes multicast prepare messages to other nodes, indicating they have already verified the
  pre-prepared message. once a node received more than 2f prepare messages from other nodes which match pre-prepared
  message, it considered itself as ready to commit, and multicast commit message.
+ commit: if a backup node has received 2f+1 commit messages, it would execute it, and multicast reply message to the
  client
+ f: maximum faulty nodes can be tolerated in the system

## EVM

### ByteCode

#### stack and opcodes
+ the stack has maximum depth of 1024
+ push1 push the byte following this opcode onto the stack. The top 31 bytes of the word are zero filled
+ CALLVALUE: push on to the stack how much wei was sent with the transaction
+ DELEGATECALL: stack[0] temp gas, stack[1] to address, stack[2] start offset for arguments in memory, stack[3] length for arguments in memory, stack[4] start offset for return in memory, stack[5] length for return in memory. The return value also set to interpreter.
+ DUP1: duplicate the top of the stack
+ DUP2: duplicate stack[1] on to the stack
+ ISZERO: pop a word off the stack, if the word is 0 push 1 onto the stack otherwise, push 0 onto the stack
+ JUMPI: jump if, set pc to stack[0] if stack[1] is not zero. Pop two values off the stack
+ JUMPDEST: jump destination
+ REVERT: halt execution and indicate a revert has occured. Use stack[0] as a memory location and stack[1] as a length of revert reason
+ POP: pop to top value off the stack
+ SSTORE: store a word(32 bytes) to storage, pop two values off the stack stack[0] is the location to write to, stack[1] is the value to write
+ CODECOPY: copy from code that is executing to memory
  stack[0]: memory offset to write to; stack[1]: code offset to read from; stack[2]: the length in bytes to copy;
  pop three values off the stack
+ RETURN: return the result, stack[0] the starting offset in memory, stack[1] the length of the result;
+ RETURNDATACOPY: copy the return value from interpreter to memory, pop three items from stack, stack[0] marks the starting offset in memory, stack[1] marks the starting offset in interpreter, stark[2] marks the length.
+ INVALID: mark the end of init code
+ CALLDATASIZE: push the size of the transaction data field onto the stack
+ LT: less then: if stack[0] < stack[1] pop two values off the stack and push 1 onto the stack, otherwise push 0 onto the stack
+ CALLDATALOAD: push onto the stack the 32 bytes(of calldata) starting at offset stack[0], all parameters are passed as 32 byte words
+ SHR: shift stack[1] to the right stack[0] times, pop stack[0] off the stack;
+ STOP: end execution, indicate execution success, and don't return any data;
+ DUP2: duplicate 2nd stack item i.e stack[1]
+ SWAP2: swap stack[0] and stack[2]
+ SLOAD: pop a word off the stack, load a word from storage located by the word.
+ SIGNEXTEND: extends a signed integer from (b + 1) * 8 bits to 256 bits, where b represents the number of bytes of the original signed integer, used when a smaller integer is upcasted to a larger type.
+ i.e: load the free memory pointer from locaton 0x40 (0x40 in memory means the position in memory that the contract can freely use)
    push1 0x40
    dup1 
    mload
+ MSTORE: store stack[1] in memory location specified by stack[0]
+ MLOAD: push on to the stack the value at memory location specified by stack[0], and pop stack[0]
+ tip: put small variables together can extremelly reduce the use of gas, for sstore opcode can be reused among these variables and save storage space, the example below uses sstore only once.
  
    | command                                    | explain                    |
    | ------------------------------------------ | -------------------------- |
    | PUSH1 0x02                                 | load storage location 0x02 |
    | DUP1                                       |
    | SLOAD                                      |
    | PUSH1 0x12                                 | mask in uint32 = 0x12      |
    | PUSH4 0xffffffff                           |
    | NOT                                        |
    | SWAP1                                      |
    | SWAP2                                      |
    | AND                                        |
    | OR                                         |
    | ------------------------------------------ |
    | PUSH12 0xf..f0..00                         | mask uint64 = 0x13         |
    | NOT                                        |
    | AND                                        |
    | PUSH5 0x1300000000                         |
    | ------------------------------------------ |
    | PUSH12 0xfff..ff                           | mask address(0x14)         |
    | AND                                        |
    | PUSH1 0x05                                 |
    | PUSH1 0x62                                 |
    | SHL                                        |
    | OR                                         |
    | ------------------------------------------ |
    | SWAP1                                      | store to location0x02      |
    | SSTORE                                     |

+ tip: there is no way to optimize storing byte and it would be cheaper to use uint256 simply

    store a byte       |                                                          | store a uint256
    ------------------------------------------------------------------------------------------------
    PUSH1 0x00         | load storage location 0x00                               |PUSH1 0x11
    DUP1               |                                                          |PUSH2 0x01
    SLOAD              |                                                          |SSTORE
    -------------------------------------------------------------------------------
    PUSH1 0x10         | push the constant we want to set the storage value to    |
    -------------------------------------------------------------------------------
    PUSH1 0xff         | create a bit mask                                        |
    NOT                |                                                          |
    -------------------------------------------------------------------------------
    SWAP1              | mask off the bottom byte in the word loaded from storage |
    SWAP2              |                                                          |
    AND                |                                                          |
    OR                 |                                                          |
    SWAP1              |                                                          |
    ------------------------------------------------------------------------------- 
    SSTORE             | store the result to location 0x00                        |
    ------------------------------------------------------------------------------------------------
#### bytes and string

+ for 0 to 31 bytes 
  
| Byte 31 | Byte 30 | Byte 29 | ... | Byte 2   | Byte 1   | Byte 0        |
---------------------------------------------------------------------------
|         Data                                            |7|6|5|4|3|2|1|0|
---------------------------------------------------------------------------
| byte[0] | byte[1] | byte[2] | ... | byte[29] | byte[30] |0|0| length  |**0**|

the last 0 in Byte0 mark that the length of the bytes is less then 32

+ for 32 bytes or more

| Byte 31 | Byte 30 | Byte 29 | ... | Byte 2   | Byte 1   | Byte 0        |
---------------------------------------------------------------------------
|                        length                                         |1|

* at storage location storage[keccak256(storage slot)]

| byte[0] | byte[1] | byte[2] | ... | byte[29] | byte[30] | byte[31] |

* at storage location storage[keccak256(storage slot) + 1]
  
| byte[32] | byte[33] | byte[34] | ... |

+ i.e opcode for the below solidity code

```solidity
contract StorageBytes {
  bytes private valBytes;
  function setBytes(byte _val) {
    valBytes.push(_val);
  }
}
```
tip: parameter is 0x0100000000000000000000000000000000000000000000000000000000000000 when calling this function

CALLDATALOAD        |  load one word of calldata from offset 0x04(0x04 already in the stack, skip the function selector)
PUSH1 0x01          |
PUSH1 0x01          |
PUSH1 0xf8          |
SHL                 |
SUB                 |
NOT                 | create a bit mask 0xff0000...00 
AND                 |
PUSH1 0x53          | 
JUMP                |

stack is:

stack[0]: 0x??0000..00
stack[1]: 0x51

SLOAD               | load from storage location 0x00 which is the length of valBytes if it is dlarger than 31 bytes
PUSH1 0x3f          | 
DUP2                |
AND                 | mask off what would be the length of the bytes, left the last bit length indicator
DUP1                |
PUSH1 0x31          | check whether the length of the bytes is exactly 31
DUP2                |
EQ                  |
PUSH1 0x84          |
JUMP                |

#### dynamic arrays

```solidity
contract StorageArrays {
  uint256 private arrayUint256;
  byte[] private arrayByte;
  function setUint256ArrayVal(uint256 _ofs, uint256 _val) external {
    arrayUint256[_ofs] = _val;
  }
  function setByteArrayVal(uint256 _ofs, byte _val) external {
    arrayByte[_ofs] = _val;
  }
}
```

values for `uint256 private arrayUint256;` stored at locations: storage[keccak256(slot number)+offset] = value

the number of elements in the dynamic array is stored at storage[slot number]

#### mappings 
```solidity
contract StorageMappings {
  mapping(uint256=>uint256) private map;

  function setMapVal(uint256 _key, uint256 _val) external {
    map[_ofs] = _val;
  }
}
```

values for `mapping(uint256=>uint256) private map` stored at locations: storage[keccak256(key.slot number)] = value

nothing is stored at storage[slot number]

#### memory

+ 0x00 - 0x3f: (64 bytes) scratch space for hashing methods
+ 0x40 - 0x5f: (32 bytes) currently allocated memory size (aka. free memory pointer)
+ 0x60 - 0x7f: (32 bytes) zero slot
+ 0x80: allocated memory start here i.e `uint[] memory a = new uint[](7)`

#### CODECOPY & EXTCODECOPY

+ CODECOPY: copy bytes from this contract to memory
+ EXTCODECOPY: copy bytes from another contract to memory
usage:
+ revert reason error messages
+ deploy a contract from this contract
+ any other static data (since CODECOPY is much cheaper than sload, static data located in code binary)
   


## Assembly & Yul

### Use case

#### proxy code

+ original code:
  ```solidity
  contract BuggyContract {
    address public impl;  // slot[0] might cause collision
    constructor(address _impl) {
      impl = _impl;
    }
    fallback() external payable {
      (bool ok, bytes memory res) = impl.delegatecall(msg.data);
      require(ok, "delegate call failed"); //not propagating result / error message
    }
  }
  ```

+ collision fix:
  ```solidity
  contract ProxyGetImpl {
    constructor(address _impl) {
      assembly {
        sstore(address(), _impl) // use present contract address as the slot, to reduce the use of gas
      }
    }
    function PROXY_getImpl() public view returns(address impl) {
      assembly {
        impl := sload(address()) // load from the slot
      }
    }
    fallback() external payable {
      assembly {

      }
    }
  }
  ```


## Userful assembly Pattern

### retrieve parameters from _calldata
if _calldata is stored in calldata you can simply use slic syntax to seperate the 4-bytes selector and parameter part
```solidity
if (bytes4(fnCallData) == ERC20.approve.selector) {
    // ABI-decode the remaining bytes of fnCallData as IERC20.approve() parameters
    // using a calldata array slice to remove the leading 4 bytes.
    (address spender, uint256 allowance) = abi.decode(fnCallData[4:], (address, uint256));
    require(isAllowedSpender[spender], 'not an allowed spender');
}
```

however if the _calldata is stored in memory, things become much tricky, you can either copy the rest of the array or you tempory modify the array in-place and pass it to abi.decode, here is example for the last option

```solidity
function retrieveParameter(bytes memory _calldata) external {
  bytes32 preserve;
  if (bytes4(_calldata) == ERC20.approve.selector) {
    assembly {
      // get the length
      let length := mload(_calldata)
      // get the position start from which need to modify
      _calldata := add(_calldata, 4)
      // preserve the old bytes
      preserve := mload(_calldata)
      // cover the selector part with length
      mstore(_calldata, sub(length, 4))
    }
    (address spender, uint256 allowance) = abi.decode(fnCallData, (address, uint256));
    require(isAllowedSpender[spender], 'not an allowed spender');
    // restore
    assembly {
      mstore(fnCallData, preserve)
      _calldata := sub(_calldata, 4)
    }
  }
  // ... rest of the function
}
```


## Common ERC

### EIP-121

### ERC-191
  ```solidity
  function signatureBasedExecution(address target, uint256 nonce, bytes memory payload, uint8 v, bytes32 r, bytes32 s) public payable {
        
    // Arguments when calculating hash to validate
    // 1: byte(0x19) - the initial 0x19 byte
    // 2: byte(0) - the version byte
    // 3: address(this) - the validator address
    // 4-6 : Application specific data

    bytes32 hash = keccak256(abi.encodePacked(byte(0x19), byte(0), address(this), msg.value, nonce, payload));

    // recovering the signer from the hash and the signature
    addressRecovered = ecrecover(hash, v, r, s);
   
    // logic of the wallet
    // if (addressRecovered == owner) executeOnTarget(target, payload);
  }
  ``` 
+ 0x19: use 0x19 to make sure the signed_data is not a valid RLP, any EIP-191 signed_data can never be an ethereum transaction
+ version 0x00, followed by <intended validator address> in the case of a Multisig wallet that perform an execution based on a passed signature, the validator address is the address of the Multisig itself. This field is to prevent the reuse of the signature on aother validator.
+ version 0x01: for the structure of eip-721, version specific data for version 0x01 is dominSeparator
  
### ERC-712
+ EIP712Domin: string name; string version; uint256 chainId; address verifying contract; bytes32 salt;
+ encodeType: i.e. Mail(address from,address to,string contents)
+ encodeData: enc(value₁) ‖ enc(value₂) ‖ … ‖ enc(valueₙ)
  true|false => uint256 0, 1; string|bytes => keccak(string|bytes); address => uint160
+ hashStruct: hashStruct(s : 𝕊) = keccak256(typeHash ‖ encodeData(s)); typeHash = keccak256(encodeType(typeOf(s)))
+ dominSeparator: domainSeparator = hashStruct(eip712Domain)
+ eth_signTypedData: sign(keccak256("\x19\x01" ‖ domainSeparator ‖ hashStruct(message)))
+ i.e.: 
  ```solidity
  DOMAIN_SEPARATOR = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes(name)),
            keccak256(bytes(version)),
            chainId,
            address(this)
        ));
  bytes32 private constant MESSAGE_TYPEHASH = keccak256("EIP712Message(uint256 value)");
  function hash(EIP712Message memory message) internal view returns (bytes32) {
        return keccak256(abi.encodePacked(
            "\x19\x01",
            DOMAIN_SEPARATOR,
            keccak256(abi.encode(
                MESSAGE_TYPEHASH,
                message.value
            ))
        ));
    }
  ```

### ERC-165
+ interface:
```solidity
pragma solidity ^0.4.20;

interface ERC165 {
    /// @notice Query if a contract implements an interface
    /// @param interfaceID The interface identifier, as specified in ERC-165
    /// @dev Interface identification is specified in ERC-165. This function
    ///  uses less than 30,000 gas.
    /// @return `true` if the contract implements `interfaceID` and
    ///  `interfaceID` is not 0xffffffff, `false` otherwise
    function supportsInterface(bytes4 interfaceID) external view returns (bool);
}
```
+ using XOR to adapt to the interface with multiple functions
```solidity
function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return
          interfaceID == this.supportsInterface.selector || // ERC165
          interfaceID == this.is2D.selector
                         ^ this.skinColor.selector; // Simpson
    }
```
+ mapping implementation: the implementation contract is reuseable and execution cost constant 2825 gas for any input
```solidity
pragma solidity ^0.4.20;

import "./ERC165.sol";

contract ERC165MappingImplementation is ERC165 {
    /// @dev You must not set element 0xffffffff to true
    mapping(bytes4 => bool) internal supportedInterfaces;

    function ERC165MappingImplementation() internal {
        supportedInterfaces[this.supportsInterface.selector] = true;
    }

    function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return supportedInterfaces[interfaceID];
    }
}

interface Simpson {
    function is2D() external returns (bool);
    function skinColor() external returns (string);
}

contract Lisa is ERC165MappingImplementation, Simpson {
    function Lisa() public {
        supportedInterfaces[this.is2D.selector ^ this.skinColor.selector] = true;
    }

    function is2D() external returns (bool){}
    function skinColor() external returns (string){}
}
```
+ pure implementation: the gas cost increases linerly with a higher number of supported interfaces
```solidity
contract Homer is ERC165, Simpson {
    function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return
          interfaceID == this.supportsInterface.selector || // ERC165
          interfaceID == this.is2D.selector
                         ^ this.skinColor.selector; // Simpson
    }

    function is2D() external returns (bool){}
    function skinColor() external returns (string){}
}
```
### EIP-155 Simple replay attack protection

### ERC-4973

### ERC-1271

### ERC-6909
+ basically similar to erc-1155, without batchTransfer and recipient callback

### ERC-1967 Proxy Storage Slots
+ logic contract address: bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)
+ admin address: bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1)
+ beacon address: bytes32(uint256(keccak256('eip1967.proxy.beacon')) - 1)
+ how to avoid function selector clash
  1. storage variables in proxy contract are located in specific location, to avoid clash
  2. proxy contract admin is not allowed to call logic contract
### EIP-1822 Universal Upgrageable Proxy Standard
+ logic contract address: keccak256("PROXIABLE")
+ the arbitrary nature of the Proxy Contract’s constructor provides the ability to select from one or more constructor functions available in the Logic Contract source code (e.g., constructor1, constructor2, … etc. )
+ proxiable contract should implement following function
  ```solidity
  contract Proxiable {
    // Code position in storage is keccak256("PROXIABLE") = "0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7"

    function updateCodeAddress(address newAddress) internal {
        require(
            bytes32(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7) == Proxiable(newAddress).proxiableUUID(),
            "Not compatible"
        );
        assembly { // solium-disable-line
            sstore(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7, newAddress)
        }
    }
    function proxiableUUID() public pure returns (bytes32) {
        return 0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7;
    }
}```
+ pitfalls when using a proxy
  1. we recommend utilizing a single “base” contract which holds all variables, and which is inherited in subsequent logic contract(s). This practice greatly reduces the chances of accidentally reordering variables or overwriting them in storage.
  2. Restricting dangerous functions: there is a time gap between the deployment of proxy contract and logic contract, to avoid abuse of logic contract functions, we should give away the owner ship as quick as possible, we can also use LibraryLock to avoid malicious call.
   ```solidity
   contract LibraryLock is LibraryLockDataLayout {
    // Ensures no one can manipulate the Logic Contract once it is deployed.
    // PARITY WALLET HACK PREVENTION

    modifier delegatedOnly() {
        require(initialized == true, "The library is locked. No direct 'call' is allowed");
        _;
    }
    function initialize() internal {
        initialized = true;
    }
  }
  contract MyToken is ERC20DataLayout, ERC20, Owned, Proxiable, LibraryLock {

    function constructor1(uint256 _initialSupply) public {
        totalSupply = _initialSupply;
        tokens[msg.sender] = _initialSupply;
        initialize();
        setOwner(msg.sender);
    }
    function updateCode(address newCode) public onlyOwner delegatedOnly  {
        updateCodeAddress(newCode);
    }
  }
   ```
