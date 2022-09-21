---
layout: post
title: More EVM Puzzles Part 1
category: EVM Puzzles
excerpt_separator:  <!--more-->
---

## Challenge 1:
The puzzles for these challenges are located in `puzzles/`. Challenge 1 contains the following:
```
{
  "code": "36340A56FEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFEFE5B58360156FEFE5B00",
  "askForValue": true,
  "askForData": true
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      36      CALLDATASIZE
01      34      CALLVALUE
02      0A      EXP
03      56      JUMP
04      FE      INVALID
05      FE      INVALID
06      FE      INVALID
07      FE      INVALID
08      FE      INVALID
09      FE      INVALID
0A      FE      INVALID
0B      FE      INVALID
0C      FE      INVALID
0D      FE      INVALID
0E      FE      INVALID
0F      FE      INVALID
10      FE      INVALID
11      FE      INVALID
12      FE      INVALID
13      FE      INVALID
14      FE      INVALID
15      FE      INVALID
16      FE      INVALID
17      FE      INVALID
18      FE      INVALID
19      FE      INVALID
1A      FE      INVALID
1B      FE      INVALID
1C      FE      INVALID
1D      FE      INVALID
1E      FE      INVALID
1F      FE      INVALID
20      FE      INVALID
21      FE      INVALID
22      FE      INVALID
23      FE      INVALID
24      FE      INVALID
25      FE      INVALID
26      FE      INVALID
27      FE      INVALID
28      FE      INVALID
29      FE      INVALID
2A      FE      INVALID
2B      FE      INVALID
2C      FE      INVALID
2D      FE      INVALID
2E      FE      INVALID
2F      FE      INVALID
30      FE      INVALID
31      FE      INVALID
32      FE      INVALID
33      FE      INVALID
34      FE      INVALID
35      FE      INVALID
36      FE      INVALID
37      FE      INVALID
38      FE      INVALID
39      FE      INVALID
3A      FE      INVALID
3B      FE      INVALID
3C      FE      INVALID
3D      FE      INVALID
3E      FE      INVALID
3F      FE      INVALID
40      5B      JUMPDEST
41      58      PC
42      36      CALLDATASIZE
43      01      ADD
44      56      JUMP
45      FE      INVALID
46      FE      INVALID
47      5B      JUMPDEST
48      00      STOP
```
</div>
</details>


<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
We are expected to send in both calldata and callvalue for this challenge. Reviewing the opcodes, we can tell that we will be jumping to the location calculated by (CALLVALUE ** CALLDATASIZE). Ideally we'd end up at the final JUMPDEST at 0x47 and call it a day but that's 71 in decimal and not a particularly friendly number to work with. So, we now have a few options for getting ourselves to the JUMPDEST at 0x40 which is 64 in decimal. Looking ahead, we see that we will be effectively pushing 0x41 onto the stack and then adding the size of our calldata to it before jumping to that location. Cool, we know we want to get to 0x47 so our calldata should be 6 bytes in size. That means our call value will be 2 wei since 2**6 = 0x40. 


```
{"value":2,"data":"0xabcdef012345"}
```

</div>
</details>

## Challenge 2:
The puzzles for these challenges are located in `puzzles/`. Challenge 2 contains the following:
```
{
  "code": "3660006000373660006000F0600080808080945AF13D600a14601F57FEFEFE5B00",
  "askForValue": false,
  "askForData": true
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      36        CALLDATASIZE
01      6000      PUSH1 00
03      6000      PUSH1 00
05      37        CALLDATACOPY
06      36        CALLDATASIZE
07      6000      PUSH1 00
09      6000      PUSH1 00
0B      F0        CREATE
0C      6000      PUSH1 00
0E      80        DUP1
0F      80        DUP1
10      80        DUP1
11      80        DUP1
12      94        SWAP5
13      5A        GAS
14      F1        CALL
15      3D        RETURNDATASIZE
16      600A      PUSH1 0A
18      14        EQ
19      601F      PUSH1 1F
1B      57        JUMPI
1C      FE        INVALID
1D      FE        INVALID
1E      FE        INVALID
1F      5B        JUMPDEST
20      00        STOP
```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
Let's break this challenge down into two blocks

### Block 1
```
00      36        CALLDATASIZE
01      6000      PUSH1 00
03      6000      PUSH1 00
05      37        CALLDATACOPY
06      36        CALLDATASIZE
07      6000      PUSH1 00
09      6000      PUSH1 00
0B      F0        CREATE
```

This block is all used to setup the CREATE opcode. All we are doing here is taking our callvalue and copying that into memory to serve as our initializiation code for the new contract we will CREATE.

### Block 2
```
0C      6000      PUSH1 00
0E      80        DUP1
0F      80        DUP1
10      80        DUP1
11      80        DUP1
12      94        SWAP5
13      5A        GAS
14      F1        CALL
15      3D        RETURNDATASIZE
16      600A      PUSH1 0A
18      14        EQ
19      601F      PUSH1 1F
1B      57        JUMPI
```

This block now makes a call to our newly created contract. Note that all of the arguments are 0 except for gas and the address. The call to the newly address will therefore contain no calldata. After the call, we get back the size in bytes of our return data and check that it is equal to 0x0a. If so, then we will take the JUMPI and complete this challenge. What we want then is to create a contract such that when an external call is made to our contract with no function data it returns data that is 10 (0x0a) bytes long. Conveniently, this can be achieved through a fallback function.

```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.16;


contract ReturnTenBytes {

    fallback(bytes calldata) external returns(bytes memory) {
        bytes10 derp = 0x01020304050607080910;
        return abi.encodePacked(derp);
    }
}
```


```
{"data":"0x608060405234801561001057600080fd5b5060eb8061001f6000396000f3fe6080604052348015600f57600080fd5b5060003660606000690102030405060708091060b01b90508060405160200160369190609c565b604051602081830303815290604052915050915050805190602001f35b60007fffffffffffffffffffff0000000000000000000000000000000000000000000082169050919050565b6000819050919050565b60966092826053565b607f565b82525050565b600060a682846089565b600a820191508190509291505056fea2646970667358221220e25716522e5bb854bfb1b889755e9b80453278e42970c53115815194240a401d64736f6c63430008100033","value":0}
```

</div>
</details>



## Challenge 3:
The puzzles for these challenges are located in `puzzles/`. Challenge 3 contains the following:
```
{
  "code": "3660006000373660006000F06000808080935AF460055460aa14601e57fe5b00",
  "askForValue": false,
  "askForData": true
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      36        CALLDATASIZE
01      6000      PUSH1 00
03      6000      PUSH1 00
05      37        CALLDATACOPY
06      36        CALLDATASIZE
07      6000      PUSH1 00
09      6000      PUSH1 00
0B      F0        CREATE
0C      6000      PUSH1 00
0E      80        DUP1
0F      80        DUP1
10      80        DUP1
11      93        SWAP4
12      5A        GAS
13      F4        DELEGATECALL
14      6005      PUSH1 05
16      54        SLOAD
17      60AA      PUSH1 AA
19      14        EQ
1A      601E      PUSH1 1E
1C      57        JUMPI
1D      FE        INVALID
1E      5B        JUMPDEST
1F      00        STOP
```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
This one is a lot like challenge 2 with two minor differences. First, we are now using a DELEGTECALL as opposed to a CALL. Second, instead of checking the size of our return data, we are checking the contents of our fifth storage slot. This is a useful combination as DELEGATECALL allows the call to our newly created contract to modify the caller's state. Thus we can solve this challenge by setting the fifth storage slot to 0xAA when our contract is called into.

```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.16;


contract StoreSlotFive {

    uint256 slotZero = 0;
    uint256 slotOne = 1;
    uint256 slotTwo = 2;
    uint256 slotThree = 3;
    uint256 slotFour = 4;
    uint256 slotFive = 5;

    fallback() external {
        slotFive = 170;
    }
}

```


```
{"data":"0x6080604052600080556001805560028055600380556004805560058055348015602757600080fd5b50604f8060356000396000f3fe6080604052348015600f57600080fd5b5060aa600581905500fea2646970667358221220dc6332a94c366877000ab52293ddc44fabb9baedc61e97b7ed4fd539d46f21cb64736f6c63430008100033","value":0}
```

</div>
</details>



## Challenge 4:
The puzzles for these challenges are located in `puzzles/`. Challenge 4 contains the following:
```
{
  "code": "30313660006000373660003031F0319004600214601857FD5B00",
  "askForValue": true,
  "askForData": true
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      30        ADDRESS
01      31        BALANCE
02      36        CALLDATASIZE
03      6000      PUSH1 00
05      6000      PUSH1 00
07      37        CALLDATACOPY
08      36        CALLDATASIZE
09      6000      PUSH1 00
0B      30        ADDRESS
0C      31        BALANCE
0D      F0        CREATE
0E      31        BALANCE
0F      90        SWAP1
10      04        DIV
11      6002      PUSH1 02
13      14        EQ
14      6018      PUSH1 18
16      57        JUMPI
17      FD        REVERT
18      5B        JUMPDEST
19      00        STOP
```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
The crux of this challenge is understanding that the CREATE opcode runs the initialization code on the newly created contract. Further, we know that initialization code always contains a contract's constructor. Therefore, we can create a contract such that the constructor performs our desired actions. But what are the desired actions?


We can see that the balance of the calling context is taken and sent to the CREATE opcode. In this case that will be the call value we are sending along. We can also see that upon successful contract creation, the balance of the new address is checked and then becomes the divisor in a DIV operation with the initial balance. This needs to equal to 2 for us to solve the puzzle. So, we get to choose any values X and Y such that X / Y = 2. X will be our callvalue and Y will be our ending balance for the newly created contract. I chose to send 4 wei to the new contract and to forward 2 of the wei to the 0 address during construction.

```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.9;


contract LowerBalanceIn {
    constructor() payable {
        payable(address(0)).send(2);
    }
}

```


```
{"value":4 "data":"0x6080604052600073ffffffffffffffffffffffffffffffffffffffff166108fc60029081150290604051600060405180830381858888f1935050505050603f8060496000396000f3fe6080604052600080fdfea2646970667358221220d67c79b5559a559287188a32474490c251553397b7209a654a3611ac156a20ef64736f6c63430008090033"}
```

</div>
</details>

## Challenge 5:
The puzzles for these challenges are located in `puzzles/`. Challenge 5 contains the following:
```
{
  "code": "60203611600857FD5B366000600037365903600314601957FD5B00",
  "askForValue": false,
  "askForData": true
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      6020      PUSH1 20
02      36        CALLDATASIZE
03      11        GT
04      6008      PUSH1 08
06      57        JUMPI
07      FD        REVERT
08      5B        JUMPDEST
09      36        CALLDATASIZE
0A      6000      PUSH1 00
0C      6000      PUSH1 00
0E      37        CALLDATACOPY
0F      36        CALLDATASIZE
10      59        MSIZE
11      03        SUB
12      6003      PUSH1 03
14      14        EQ
15      6019      PUSH1 19
17      57        JUMPI
18      FD        REVERT
19      5B        JUMPDEST
1A      00        STOP

```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
This challenge doesn't require creating a contract and is quite straightforward. We need to supply calldata greater in length than 0x20 to jump over the first `REVERT` instruction. Then, we need our calldata to be 3 bytes less in size than what is in memory. Huh, but aren't we placing the calldata in memory? The key here is that `MSIZE` reads memory in 0x20 byte increments. So we just need to place in any calldata that is 3 less than a multiple of 0x20 (and greater than 0x20). We can choose 0x40 for the multiple and submit calldata of length 0x3d.


```
{"data":"0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff","value":0}
```

</div>
</details>

