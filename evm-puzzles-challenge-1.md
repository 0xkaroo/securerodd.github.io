# EVM Codes Write-ups

This series will be dedicated to write-up solutions for the evm-puzzles by <Attribution>. Each write-up will contain progressively more information hidden behind 'hints'. The hope is to reduce excessive spoilers and give only as much info as is needed for those who would like to solve the puzzles.

## Setting up the environment:

1. first things first, clone the repo at https://github.com/fvictorio/evm-puzzles. This contains all of the challenges we'll be going through to improve our understanding of EVM byetecode.
2. Get familiar with the tools we'll be using by reading through the hardhat docs <LINK> and playing around with [evm.codes](https://www.evm.codes)
3. Optional: Read and walkthrough 1st portion of noxx's EVM series

### Challenge 1:
Our first challenge is located in /puzzles/puzzle_1.json. Challenge 1 contains the following:
```
{
  "code": "3456FDFDFDFDFDFD5B00",
  "askForValue": true,
  "askForData": false
}
```
<details>
<summary>Hint 1:</summary>
<br>
The "code" section contains EVM bytecode. Try converting that to opcodes (one way is to drop the bytecode into https://www.evm.codes/playground) 

Hint 2:
The corresponding EVM opcodes are:

CALLVALUE	
JUMP	
REVERT	
REVERT	
REVERT	
REVERT	
REVERT	
REVERT	
JUMPDEST	
STOP

The goal of this challenge is to not cause a revert.
</details>


### Hint 3:
Callvalue does X and Jump does Y. How can we Jump over the 6 Revert opcodes and land on the Jump destination?


### Hint 4:
Each of the Opcodes are 1 byte in size

### Hint 5:
CALLVALUE //Offset 0	
JUMP	  //Offset 1
REVERT    //Offset 2	
REVERT    //Offset 3	
REVERT    //Offset 4	
REVERT    //Offset 5	
REVERT    //Offset 6	
REVERT    //Offset 7	
JUMPDEST  //Offset 8
STOP

### Solution:
Pass in a value of 8 wei, so that callvalue puts 8 on thestack and JUMP jumps to the JUMPDEST at the offset 8.


