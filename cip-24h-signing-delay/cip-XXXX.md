## Title

<pre>
  CIP: CIP XXXX
  Layer: Daml
  Title: 24h Submission Delay for CC Token Standard Implementation
  Author: Simon Meier, Moritz Kiefer
  Status: Draft
  Type: Standards Track
  Created: 2026-01-15
  License: CC0-1.0
</pre>

## Abstract

The initial Token Standard (CIP 56) implementation of Canton Coin included a limitation that only a 10 minute submission delay between preparing and executing
was supported. This CIP lifts this limitation to support a 24h delay as targeted by CIP 56. In addition to the token standard implementation,
the 24h submission delay will also be supported for traffic purchases and DevNet tap.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Specification

### High Level Summary

The current 10 minute submission delay comes from the use of
`OpenMiningRound` contracts. To support a 24h submission delay a new
`ExternalPartyConfigState` contract is introduced that is active for
48h and copies the subset of information required for token standard
operations from the latest `OpenMiningRound` contract at the time of
its creation.

### Detailed Changes

- Add a new configuration parameter `externalPartyConfigStateTickDuration` which defaults to 24h.
- Introduce a new `ExternalPartyConfigState` template which stores the
  amulet price, the round number, the holding fees and `maxNumInputs`,
  `maxNumOutputs`, `maxNumLockHolders` copied from the
  `OpenMiningRound` contract at the time of creation. Each of these
  contracts is active for `2 * externalPartyConfigStateTickDuration` and a new one is created every `externalPartyConfigStateTickDuration`.
- Change the token standard implementation of CC to rely on
  `ExternalPartyAmuletRules` and `ExternalPartyConfigState` instead of `AmuletRules` and
  `OpenMiningRound`. At any given point the most recent
  `ExternalPartyConfigState` contract is still valid for at least
  `externalPartyConfigStateTickDuration`, i.e., 24h with the default
  configuration providing the desired submission delay. New `Amulet` contracts created from those choices will
  have the round they were created in set to the round from `ExternalPartyConfigState`.
- Change the transfer implementation to create `FeaturedAppActivityMarker` contracts instead of directly creating `AppRewardCoupon`. This is required
  as `AppRewardCoupon`s are tied to a round which would impose a shorter submission delay. This does not change the amount of rewards that can be minted. However it does imply that `extraFeaturedAppRewardAmount` can no longer be configured independently of `featuredAppActivityMarkerAmount`, which is a feature that was never made use of.
- Enforce additional restriction on the `AmuletConfig`. These restrictions all hold already on DevNet, TestNet and MainNet so this is not changing the configuration.
  - CC usage fees, which were set to zero in CIP 78, can no longer be set to non-zero values.
  - `extraFeaturedAppRewardAmount` must be identical to `featuredAppActivityMarkerAmount`.
  - The `futureValues` field in the configuration schedule must be empty. The options for setting it were already removed
    in CIP 51. Now it is just enforced more directly.
- Expose choices for `AmuletRules_BuyTraffic` and `AmuletRules_Tap`
  through two new interface packages `splice-api-tap-v1` and
  `splice-api-traffic-purchase-v1`. These will also rely on `ExternalPartyConfigState` instead of `OpenMiningRound`. Note that during the implementation this may be rolled as a separate step.
- Add new `LockedAmulet_UnlockV2` and `LockedAmulet_OwnerExpireLockV2` choices that do not require an `OpenMiningRound`.
- Remove the existing `LockedAmulet_Unlock` and `LockedAmulet_OwnerExpireLock` choices.
- Deprecate the non-token standard `TransferCommand` template and corresponding validator endpoints so they can be removed
  in future versions. This also applies to the corresponding validator APIs `/v0/admin/external-party/transfer-preapproval/prepare-send`
  and `/v0/admin/external-party/transfer-preapproval/submit-send`.

## Motivation

Between preparing and executing a transaction, the key(s) registered in the topology state
for the submitting external party must sign the transaction. This
often requires explicit human approval sometimes even from multiple
people. Doing that within a 10 minute window can be quite disruptive
and does not match the expectations from other networks or even other
assets on the global synchronizer. Supporting a 24h delay aligns
Canton Coin with other CIP 56 assets.

## Rationale

### Propagation Delay

Supporting a longer submission delay necessarily implies that the
accessed contracts need to be active for at least the duration of the
submission delay.  This is what the new `ExternalPartyConfigState`
contract accomplishes. 

This does however imply that changes to the
values stored on that contract propagate more slowly. More specifically it
takes up to 48h until the old values cannot be used anymore.

For `maxNumInputs`, `maxNumOutputs`, `maxNumLockHolders` this is a
non-issue as those values have not been changed once and we don't
expect to need any quick changes. The `amuletPrice` in a CC transfer
is only used for converting the holding fee and from USD to CC. For holding fees,
minor fluctuations play a negligible role so price changes propagating
slower has negligible impact.

There is a bigger impact on traffic purchases where a price change is
more significant. There is not really any way to enforce use of recent
prices and support a long submission delay though so this is a
necessary tradeoff.

### Holding Fees

`Amulet` contracts created via choices relying on
`ExternalPartyConfigState` will have the round they were created in
set to the round at the point in time when `ExternalPartyConfigState`
was created. This can be up to 48h in the past at the time the
transaction gets executed.

The impact of this is that holding fees are also computed from that
round up to 48h in the past or in other words coins expire up to 2
days earlier after execution than they do at the moment. This means
that coins worth less than 2 days of holding fees will be expired
immediately after creation. Apart from that it has minor impact as CIP
78 changed holding fees to not be charged on transfer so coins used
before their expiration never get charged any holding fees.

### Additional AmuletConfig Restrictions

The additional restrictions enforced on `AmuletConfig` are required to
make the implementation feasible. In particular switching from creating `AppRewardCoupon` contracts to `FeaturedAppActivityMarker` without changing the issued rewards requires
that a CC transfer only produces featured application rewards, implied by no fees, and
`extraFeaturedAppRewardAmount` must be identical to `featuredAppActivityMarkerAmount`.

## Limitations

This change supports a 24h submission delay for token standard
operations, traffic purchases and tap.  However, reward minting is
still subject to the 10 minute submission delay. Short term, the
minting delegation introduced by CIP 96 allows to delegate minting to
a party that can support a shorter submission delay. Longer term, we
expect that as part of switching to traffic based rewards we will also
be able to support a longer signing delay for reward minting.

## Backwards compatibility

### Token Standard APIs

Changing the CC implementation of the token standard APIs to use
`ExternalPartyConfigState` is an implemenation-internal change.  Users of the token
standard APIs do not need to make any change in their application.
Note however, that the choice context returned by Canton Coin Scan
will change. As applications should treat this opaquely and just pass
it along, this should still not require changes in applications.

### Locking APIs

Applications that directly use `LockedAmulet_Unlock` or
`LockedAmulet_OwnerExpireLock` will need to switch to the `V2` choices
when they recompile their code against the new amulet versions.  Note
that existing DARs compiled against the previous version will continue
working so the upgrade can be done at the developer's preferred
schedule.

### Transaction History Parsing

App providers that parse transaction history through the [token standard
API](https://docs.sync.global/app_dev/token_standard/index.html#reading-and-parsing-transaction-history-involving-token-standard-contracts)
no change is required.

If you parse the CC specific choices directly you will need to adjust
your parser. In particular, `TransferPreapproval_SendV2` and the token
standard implementation of transfers and allocations no longer
exercise `AmuletRules_Transfer` internally and instead inline the
adjusted implementation of that choice to rely on
`ExternalPartyConfigState`.

## Reference implementation

A draft PR for the Daml changes can be found on [Github](https://github.com/hyperledger-labs/splice/pull/3487).
