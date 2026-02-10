# ethernaut notes

quick writeups for each level. keeping track of what i learned.

## level 1 - fallback
**bug:** receive() function changes owner if you've ever contributed any amount  
**exploit:** contribute a tiny amount first, then send eth directly to trigger receive(), now you're owner and can withdraw  
**fix:** don't put ownership logic in receive() lol

## level 2 - fallout
**bug:** "constructor" is misspelled as Fal1out so it's just a public function  
**exploit:** anyone can call Fal1out() and become owner  
**fix:** use the actual constructor() keyword (or just don't typo)

## level 3 - coinflip
**bug:** using blockhash for randomness is predictable  
**exploit:** write a contract that computes the same blockhash value and calls flip() with the correct guess. run it 10 times (once per block)  
**fix:** use commit-reveal scheme or Chainlink VRF for real randomness

## level 4 - telephone
**bug:** contract uses tx.origin for authentication instead of msg.sender  
**exploit:** call changeOwner through an intermediary contract. your EOA is tx.origin but the contract is msg.sender  
**fix:** always use msg.sender for auth, never tx.origin

## level 5 - token
**bug:** solidity 0.6 doesn't have overflow protection  
**exploit:** transfer more tokens than your balance. subtraction underflows and wraps to a huge number  
**fix:** check balances before math or just use solidity 0.8+

## level 6 - delegation
**bug:** fallback does delegatecall to whatever library address is stored  
**exploit:** call the contract with pwn() function signature. fallback catches it, delegatecalls to Delegate, which runs pwn() in Delegation's context and overwrites owner  
**fix:** don't delegatecall to untrusted code, especially in fallback

## level 7 - force
**bug:** contract assumes it can prevent receiving ETH by not having payable functions  
**exploit:** selfdestruct another contract with target address. force-sends ETH regardless  
**fix:** never rely on address(this).balance == 0 as a security check

## level 8 - vault
**bug:** "private" variables are still stored on-chain and readable  
**exploit:** read storage slot 1 directly with web3 to get the password, then call unlock()  
**fix:** don't store secrets on a public blockchain (encrypt off-chain if needed)

## level 9 - king
**bug:** uses transfer() to send prize to previous king when new king takes over  
**exploit:** become king with a contract that has no receive/fallback. when someone tries to claim kingship, transfer fails and reverts  
**fix:** use pull payment pattern or handle failed sends gracefully

## level 10 - reentrancy
**bug:** classic reentrancy - external call happens before balance update  
**exploit:** donate some eth, then withdraw. in your receive() hook, call withdraw again before the first one finishes. drain the contract  
**fix:** checks-effects-interactions pattern + ReentrancyGuard

## level 11 - elevator
**bug:** contract calls isLastFloor() twice and assumes same result both times  
**exploit:** implement a Building that toggles its return value. first call returns false, second returns true  
**fix:** cache the result or just don't trust external contracts to be pure

## level 12 - privacy
**bug:** same as vault - private storage is readable  
**exploit:** calculate storage layout (data[2] is at slot 5), read it with web3.eth.getStorageAt, extract first 16 bytes for the key  
**fix:** don't store secrets on-chain

## level 13 - gatekeeper one
**bug:** three gates - tx.origin check, gasleft mod check, and bytes8 key requirements  
**exploit:** call from a contract (tx.origin != msg.sender), craft the key from your address, brute force the gas offset until gasleft % 8191 == 0  
**fix:** don't use tx.origin or gas-based checks for authentication

## level 14 - gatekeeper two
**bug:** requires extcodesize == 0 (caller has no code) but also needs XOR calculation  
**exploit:** call from constructor (extcodesize is 0 during construction), calculate the XOR key from your contract address  
**fix:** don't use extcodesize for authentication

## level 15 - naught coin
**bug:** transfer() is locked for 10 years but transferFrom() isn't protected  
**exploit:** approve yourself for all tokens, then use transferFrom to move them  
**fix:** override both transfer and transferFrom if you want time locks

## level 16 - preservation
**bug:** delegatecall to library with mismatched storage layout  
**exploit:** call setFirstTime with your malicious contract address (overwrites slot 0). then call again and your contract's setTime() overwrites owner slot  
**fix:** match storage layouts when using delegatecall, or avoid it entirely

## level 17 - recovery
**bug:** contract addresses are deterministic (CREATE opcode)  
**exploit:** calculate the lost contract address using creator address + nonce. then call destroy() on it  
**fix:** don't leave selfdestruct in contracts with funds, or keep better records

## level 18 - magic number
**goal:** return 42 using minimal bytecode (10 bytes or less)  
**exploit:** write raw EVM bytecode. runtime: 602a60005260206000f3 (returns 42)  
**lesson:** understanding EVM opcodes is useful for optimization and attacks

## level 19 - alien codex
**bug:** array.length-- underflows, making array "length" = 2^256-1  
**exploit:** now you can access any storage slot. calculate which index maps to slot 0 (owner), use revise() to overwrite it  
**fix:** use solidity 0.8+ or check array bounds

## level 20 - denial
**bug:** partner.call{value: x}("") forwards all available gas  
**exploit:** set partner to a contract with infinite loop in receive(). withdraws fail because owner's withdraw runs out of gas  
**fix:** use pull payments, limit gas on external calls, or handle failures

## level 21 - shop
**bug:** price() is called twice - once to check, once to charge  
**exploit:** return 100 on first call (when !isSold), return 0 on second call (when isSold)  
**fix:** cache the price or don't depend on external view functions for critical logic

## level 22 - dex
**bug:** swap math is broken - doesn't use constant product formula  
**exploit:** swap back and forth between token1 and token2. each swap gets you more because the formula is wrong. eventually drain one token  
**fix:** use x * y = k constant product formula like Uniswap

## level 23 - dex two
**bug:** no whitelist on tokens - you can swap any ERC20  
**exploit:** deploy your own token, give DEX some of it, swap it for the real tokens at manipulated rates  
**fix:** whitelist which tokens can be swapped

## level 24 - puzzle wallet
**bug:** proxy storage collision + multicall allows double-spending  
**exploit:** proposeNewAdmin overwrites owner in implementation. addToWhitelist yourself. use multicall to deposit twice with same ETH. drain contract. setMaxBalance to become admin  
**fix:** use proper proxy storage layout (EIP-1967) and prevent multicall nesting

## level 25 - motorbike
**bug:** UUPS proxy implementation isn't initialized  
**exploit:** initialize the Engine directly (not through proxy), then upgradeToAndCall to a contract with selfdestruct  
**fix:** disable initializers in implementation contracts (_disableInitializers)

## level 26 - double entry point
**bug:** legacy token's delegateTransfer allows sweeping from vault  
**exploit:** write a Forta detection bot that watches for delegateTransfer calls originating from the vault and raises alert  
**lesson:** monitoring/detection is part of security

## level 27 - good samaritan
**bug:** logic flow depends on catching a specific error signature  
**exploit:** revert with NotEnoughBalance() during the 10-coin transfer. contract catches it and sends all coins instead  
**fix:** don't use try/catch error types for business logic

## level 28 - gatekeeper three
**bug:** construct0r typo makes it callable, timestamp check, balance check  
**exploit:** call construct0r to become owner, call getAllowance to pass timestamp, send ETH so contract has balance, call enter  
**fix:** use proper constructor, simplify auth logic

## level 29 - switch
**bug:** modifier checks calldata selector but position is manipulable  
**exploit:** craft calldata where offset points past the real selector. modifier sees turnSwitchOff but flipSwitch actually runs  
**fix:** don't manually parse calldata for security checks

## level 30 - higher order
**bug:** assembly does calldataload into uint8 but reads full 32 bytes  
**exploit:** send calldata with treasury value > 255. assembly reads it but uint8 truncation doesn't happen until after check  
**fix:** don't mix assembly and typed variables without understanding type casting

## level 31 - stake
**bug:** low-level token calls don't check return value + unstake doesn't clear staker mapping when balance hits 0  
**exploit:** fake-stake WETH (call returns false but not checked), selfdestruct to force ETH into contract, unstake to 0 but remain in stakers mapping  
**fix:** use SafeERC20, check return values, clear mappings properly

## level 32 - impersonator
**bug:** ECDSA signatures are malleable - (r, s) and (r, -s mod n) are both valid  
**exploit:** use one signature, then flip s value (n - s) and flip v (27 <-> 28) to get second valid signature  
**fix:** enforce low-s values (s < n/2) and track message hashes not signature bytes

## level 33 - magic animal carousel
**bug:** bit packing is inconsistent - long name can overflow into nextCrateId  
**exploit:** create crate with name long enough to overflow and overwrite nextCrateId bits  
**fix:** enforce string length limits, use proper masking for packed data

## level 34 - bet house
**bug:** withdraw uses stored deposit amounts even after underlying tokens moved  
**exploit:** wrap token, transfer/burn the wrapped position, withdraw using stale deposit value, re-deposit  
**fix:** track deposits accurately when tokens are transferred or burned

## level 35 - elliptic token
**bug:** permit uses wrong hash/domain separator  
**exploit:** craft permit signature that can be reused because hash construction is broken  
**fix:** follow EIP-712 spec exactly, use correct domain separator

## level 36 - cashback
**bug:** authorization relies on delegation bytecode assumptions  
**exploit:** abuse delegation/nonce mechanics to bypass auth and mint reward tokens  
**fix:** don't authenticate based on bytecode presence or delegation patterns

## level 37 - impersonator two
**bug:** signatures not bound to caller context, only nonce ordering  
**exploit:** replay the owner's signatures in correct nonce order to set yourself as admin then unlock  
**fix:** use EIP-712 with caller in message hash, track message hashes not just nonces

## level 38 - unique nft
**bug:** tx.origin check + receiver validation allows reentrancy  
**exploit:** use contract with owner() returning EOA to bypass tx.origin, reenter during mint callback to mint multiple times  
**fix:** no tx.origin checks, use reentrancy guard, implement proper safeMint

## level 39 - forger
**bug:** replay prevention checks signature bytes, but EIP-2098 compact sigs have different encoding  
**exploit:** mint with 65-byte signature, then mint again with same signature in 64-byte compact form (different bytes, same validity)  
**fix:** track message hashes, normalize signatures before checking

## level 40 - not optimistic portal
**bug:** executes messages before verification + message hash skips last item  
**exploit:** append malicious message as last item (not in hash), execute before finalization period  
**fix:** verify first, include all messages in hash, use checks-effects-interactions