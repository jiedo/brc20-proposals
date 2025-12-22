# BRC-20 Swap Module Balance Refund Plan

## Overview

To facilitate the shutdown of the BRC-20 Module-based Swap module on Bitcoin mainnet, this proposal aims to refund all assets deposited into the module back to users. The following describes a straightforward implementation plan.

## Motivation

The BRC-20 Swap module on Bitcoin mainnet has not achieved broad protocol consensus and has been inactive for an extended period, no longer providing value to users. To ensure that existing users can cleanly exit and recover their assets, it is reasonable to close the Swap module and refund the corresponding assets at the protocol level. This approach is fair to module users and does not harm the interests of the broader BRC-20 community.

## Method

The total amount of assets to be refunded has already been fully indexed by the existing BRC-20 indexers, i.e., stored in the OP_RETURN addresses associated with the module. Before processing any BRC-20 events at a specified block height, all BRC-20 asset balances currently held in the module address will be transferred in a single operation to the module deployer’s address. This will generate multiple asset transfer records, which are not related to inscriptions but only to the block height or Coinbase transaction.

After the refund, the balance of all tickers in the module address will be zero. Any subsequent transfers of BRC-20 assets to this address will not trigger an automatic refund.
The module deployer will be responsible for manually distributing the specific assets to users thereafter.

## Indexer Implementation

Existing indexers can support the refund process without executing any Swap module logic.
The content following OP_RETURN in the module address is the byte-reversed deployment transaction ID. Anyone can verify the deployer’s address by inspecting the deployment transaction. This verification process does not require implementation in the indexer code; only hardcoded addresses are needed.

* Bitcoin Mainnet

Before processing any BRC-20 events at block height **932,888**, first transfer all 23 types of BRC-20 assets from address
`6a208cbc2ac9896cac98d304aa42f43b98208ce8ae31c25e48d84ee852834a1a8066`
to the module deployer address:
`bc1qrj03km5h9ag24cpynyn3l5tvny9cyd0x0le42u`.

* Bitcoin Signet

Before processing any BRC-20 events at block height **283,888**, first transfer all 4 types of BRC-20 assets from address
`6a2031152ea2364a6dbaab9013be8d656a34256d7e89f0c30714bbb04a9d85e63e5b`
to the module deployer address:
`tb1qkrewl9zclku2qngth52eezdyrwmjpcspttdypa`.

## References

BRC-20 Protocol
BRC-20 Module Proposal
