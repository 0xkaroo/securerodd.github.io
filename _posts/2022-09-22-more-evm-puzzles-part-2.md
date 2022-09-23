---
layout: post
title: More EVM Puzzles Part 2
category: EVM Puzzles
excerpt_separator:  <!--more-->
---

## Challenge 6:
The puzzles for these challenges are located in `puzzles/`. Challenge 6 contains the following:
```
{
  "code": "7ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff03401600114602a57fd5b00",
  "askForValue": true,
  "askForData": false
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0      PUSH32 FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0
21      34                                                                      CALLVALUE
22      01                                                                      ADD
23      6001                                                                    PUSH1 01
25      14                                                                      EQ
26      602A                                                                    PUSH1 2A
28      57                                                                      JUMPI
29      FD                                                                      REVERT
2A      5B                                                                      JUMPDEST
2B      00                                                                      STOP
```

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
This is simply an integer overflow. We need to add 0x10 to the value on top of the stack to make it 0x01. Thus, our callvalue simply has to be 17.


```
{"value":17,"data":"0x"}
```

</div>
</details>

## Challenge 7:
The puzzles for these challenges are located in `puzzles/`. Challenge 7 contains the following:
```
{
  "code": "5a345b60019003806000146011576002565b5a90910360a614601d57fd5b00",
  "askForValue": true,
  "askForData": false
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      5A        GAS
01      34        CALLVALUE
02      5B        JUMPDEST
03      6001      PUSH1 01
05      90        SWAP1
06      03        SUB
07      80        DUP1
08      6000      PUSH1 00
0A      14        EQ
0B      6011      PUSH1 11
0D      57        JUMPI
0E      6002      PUSH1 02
10      56        JUMP
11      5B        JUMPDEST
12      5A        GAS
13      90        SWAP1
14      91        SWAP2
15      03        SUB
16      60A6      PUSH1 A6
18      14        EQ
19      601D      PUSH1 1D
1B      57        JUMPI
1C      FD        REVERT
1D      5B        JUMPDEST
1E      00        STOP
```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
This one is the first challenge that contains a loop! Here is what all of this code does. First, it places the current gas on the top of the stack. Then, it creates a loop that it will go through `CALLDATASIZE` times. Finally, once it exits the loop it will check that we have consumed precisely 0xa6 amount of gas. If we have we win! Otherwise, we fail. The pseudocode below demonstrates an interpretation of the puzzle's path:

```
gas_num = GAS;
i = CALLDATASIZE;
while (i != 0) {
    i--;
}
if ((gas_num - GAS) == 0xa6) {
    STOP
} else {
    REVERT
}
```
Ok, great so now we just need to exhaust 0xa6 in gas from the time the first `GAS` is called. 

Here are the costs of the OPCODES
```
CALLVALUE: 2
<LOOP>
JUMPDEST: 1
PUSH1 01: 3
SWAP1: 3
SUB: 3
DUP1: 3
PUSH1 00: 3
EQ: 3
PUSH1 11: 3
JUMPI: 10
PUSH1 02: 3
JUMP: 8
<END LOOP>
JUMPDEST: 1
GAS: 2
```
So we have 2 + (x(43)-11) + 3 = 166

solving for x we get 4 as our `CALLVALUE`.

```
{"value":4,"data":"0x"}
```

</div>
</details>



## Challenge 8:
The puzzles for these challenges are located in `puzzles/`. Challenge 8 contains the following:
```
{
  "code": "341519600757fd5b3660006000373660006000f047600060006000600047865af1600114602857fd5b4714602f57fd5b00",
  "askForValue": false,
  "askForData": true
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      34        CALLVALUE
01      15        ISZERO
02      19        NOT
03      6007      PUSH1 07
05      57        JUMPI
06      FD        REVERT
07      5B        JUMPDEST
08      36        CALLDATASIZE
09      6000      PUSH1 00
0B      6000      PUSH1 00
0D      37        CALLDATACOPY
0E      36        CALLDATASIZE
0F      6000      PUSH1 00
11      6000      PUSH1 00
13      F0        CREATE
14      47        SELFBALANCE
15      6000      PUSH1 00
17      6000      PUSH1 00
19      6000      PUSH1 00
1B      6000      PUSH1 00
1D      47        SELFBALANCE
1E      86        DUP7
1F      5A        GAS
20      F1        CALL
21      6001      PUSH1 01
23      14        EQ
24      6028      PUSH1 28
26      57        JUMPI
27      FD        REVERT
28      5B        JUMPDEST
29      47        SELFBALANCE
2A      14        EQ
2B      602F      PUSH1 2F
2D      57        JUMPI
2E      FD        REVERT
2F      5B        JUMPDEST
30      00        STOP
```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
Looking at the opcodes, we can see that we create a contract using whatever our calldata is then we get the balance of the calling account (which should be 0). Next, we call into our newly created contract with no value and no calldata and we need this to return 1 (no revert). Finally, we make sure we still have the same balance on this account (which started as 0).

To be honest I'm a bit thrown off by what the point of this level is. We can just send in basically anything to solve this.

```
{"data":"0x00","value":0}
```

</div>
</details>



## Challenge 9:
The puzzles for these challenges are located in `puzzles/`. Challenge 9 contains the following:
```
{
  "code": "34600052602060002060F81C60A814601657FDFDFDFD5B00",
  "askForValue": true,
  "askForData": false
}
```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      34        CALLVALUE
01      6000      PUSH1 00
03      52        MSTORE
04      6020      PUSH1 20
06      6000      PUSH1 00
08      20        SHA3
09      60F8      PUSH1 F8
0B      1C        SHR
0C      60A8      PUSH1 A8
0E      14        EQ
0F      6016      PUSH1 16
11      57        JUMPI
12      FD        REVERT
13      FD        REVERT
14      FD        REVERT
15      FD        REVERT
16      5B        JUMPDEST
17      00        STOP
```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
What we're doing here is taking the `keccak256()` hash of our callvalue input and seeing if it begins with `a8`. If it does then we win the challenge. This can be solved by brute forcing a valid solution. 

```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.9;

contract solveKeccak {
    function getDesiredCalldata() public view returns (uint256){
        for (uint i; i < 1000; i++){
            bytes32 hash = keccak256(abi.encodePacked(i));
            if (bytes1(hash) == 0xa8) {
                return i;
            }
        }
    }
}

```


```
{"value":47,"data":"0x"}
```

</div>
</details>

## Challenge 10:
The puzzles for these challenges are located in `puzzles/`. Challenge 10 contains the following:
```
{
  "code": "602060006000376000517ff0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f01660206020600037600051177fabababababababababababababababababababababababababababababababab14605d57fd5b00",
  "askForValue": false,
  "askForData": true
}

```

<details>
<summary> Opcodes: </summary>
<br>
<div markdown="1">
```
00      6020                                                                    PUSH1 20
02      6000                                                                    PUSH1 00
04      6000                                                                    PUSH1 00
06      37                                                                      CALLDATACOPY
07      6000                                                                    PUSH1 00
09      51                                                                      MLOAD
0A      7FF0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0      PUSH32 F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0F0
2B      16                                                                      AND
2C      6020                                                                    PUSH1 20
2E      6020                                                                    PUSH1 20
30      6000                                                                    PUSH1 00
32      37                                                                      CALLDATACOPY
33      6000                                                                    PUSH1 00
35      51                                                                      MLOAD
36      17                                                                      OR
37      7FABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABAB      PUSH32 ABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABABAB
58      14                                                                      EQ
59      605D                                                                    PUSH1 5D
5B      57                                                                      JUMPI
5C      FD                                                                      REVERT
5D      5B                                                                      JUMPDEST
5E      00                                                                      STOP

```
</div>
</details>

<details>
<summary> Solution:</summary>
<br>
<div markdown="1">
The final challenge! Basically, the first 32 bytes of our calldata will be part of an `AND` operation with `0xF0F0F0...`. The reulst of this will then be part of an `OR` operation with the next 32 bytes of our calldata. We want the result to be equal to `ABABAB...` to solve the puzzle. The `AND` operation will set everything the same as it comes in for odd positioned characters and flip the even positioned characters to 0. But we then get to send the result of this to the `OR` operation. So, we can just send `0xABABAB...` for 64 bytes and call it a day.

```
{"data":"0xabababababababababababababababababababababababababababababababababababababababababababababababababababababababababababababababab","value":0}
```

</div>
</details>

