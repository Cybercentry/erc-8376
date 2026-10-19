# ERC-8376: Token Launch Abuse Detection and Remediation

A buyer at a token launch cannot see what the deployer kept, whether liquidity
is locked, which privileged functions survived, or whether holders can sell.
All of it is already on chain. None of it is comparable between one launch and
the next, and none of it is readable at the moment a buyer pays.

ERC-8376 fixes the format: **twelve signals describing what the deployer did,
scored 0 to 100, with confidence reported separately, published as a report
whose evidence recomputes from chain state.** A venue reads it with one
`STATICCALL` before a purchase completes.

- **Standard:** [ERC-8376](https://github.com/ethereum/ERCs/blob/master/ERCS/erc-8376.md), merged into `ethereum/ERCs`
- **Discussion:** [Ethereum Magicians](https://ethereum-magicians.org/t/erc-8376-token-launch-abuse-detection-and-remediation/29359)
- **Licence:** CC0-1.0

## Why

Between 1 January and 17 October 2026, **614,684 funded token launches** opened
on Arbitrum, Base, Ethereum, Optimism and Robinhood Chain. In **236,215 of
them** the party that supplied the liquidity removed it again, taking
**$1,027,172,010** more out than it put in.

No buyer could read a score beforehand.

That figure comes from **one** of the twelve signals, `liquidityRemoved`. It is
a floor, not an estimate: a supplier who withdraws from a different account
than it deposited with is not counted at all.

<details>
<summary>How it was measured, and how to check it</summary>

Window 1 January to 17 October 2026, five chains, measured two independent ways
and reconciled.

| | |
|---|---|
| Pools opened | 1,348,661 (Etherscan) · 1,356,059 (Dune), 0.23% apart |
| Ever funded | 614,684 |
| Supplier removed the liquidity | 236,215 (38.4%) |
| Taken above what was put in | $1,027,172,010 |

Three defences, each answering a failure found in the data rather than imagined:

1. **Amounts come from token transfers, never from the event payload.** One
   pool emitted 36 forged `Mint` logs claiming $5.6 quadrillion each. A log
   costs one transaction to forge; moving USDC does not. The `Mint` log is used
   only to identify which transaction added liquidity.
2. **The pool contract must be deployed inside the window.** A contract can
   emit `PairCreated` naming an address that already exists. The 2020
   USDC/WETH pair entered an early census that way and contributed $183bn of
   real but irrelevant deposits.
3. **A launch pairs a new asset against an established one.** Pools whose both
   sides were priced before 2026 are infrastructure, not launches, and are
   excluded. "Established" is derived from whether a token had a price in
   December 2025, not hardcoded, because Ethereum alone carries dozens of fake
   tokens using the symbol `USDC`.

Cross-checks, none tuned to agree:

- Conservation of a pool's balance says the supplier's net gain equals net
  trader payments. Tested on 1,297 pools by comparing token transfers against
  swap logs: **median ratio 1.0000**, 5th percentile 1.0000, 95th 1.0012.
- 225 named pools compared across three instruments (event payloads, token
  transfers, swap logs priced independently): **95.1% agree within 10%**.
- Wallet-splitting understatement: **1.71%** of trader money in the sample sits
  on pools the same-actor test misses entirely.

**Conduct, not intent.** ERC-8376 excludes price decline by rule, so this
measures the act and names the actor. It cannot distinguish a rug from a
legitimate liquidity migration, and neither can the standard, which is why
remediation requires a claim and an adjudicator rather than a score alone. The
correct phrasing is "the supplier removed the liquidity", never "rugged",
"stolen" or "scammed".

</details>

## The twelve signals

Each is a deployer action, measured against a published activation threshold,
reported as unavailable rather than clean when it cannot be established.

| Signal | What it measures |
|---|---|
| `deployerSupplyShare` | share of supply the deployer kept |
| `insiderAllocationShare` | share distributed to connected addresses before launch |
| `sniperConcentration` | concentration of the earliest buyers |
| `lpLockedShare` | share of liquidity locked |
| `lpLockRemaining` | time remaining on the lock |
| `liquidityRemoved` | share of liquidity withdrawn |
| `deployerSellRatio` | share of the deployer's holding sold |
| `proceedsWithdrawnShare` | share of escrowed proceeds withdrawn |
| `privilegedPowers` | mint, pause, blacklist, fee, upgrade, seize, limits, exempt |
| `priorUpheldClaims` | upheld claims against the same deployer |
| `supplyInflation` | supply minted after launch |
| `washTradeRatio` | share of volume that is self-dealing |

Two of them, `proceedsWithdrawnShare` and `priorUpheldClaims`, **cannot be
derived by anyone today**, because they read a venue's escrow and a claims
registry that the standard itself creates. That is a property of the problem,
not an omission.

`privilegedPowers` is a bitmask: `0x0001` Mint · `0x0002` Pause · `0x0004`
Blacklist · `0x0008` Fee · `0x0010` Upgrade · `0x0020` Seize · `0x0040` Limits ·
`0x0080` Exempt.

## Reading a score

Consumer conformance is one `STATICCALL` and deploys no contract:

```solidity
interface ILaunchGuard {
    function checkLaunch(address token)
        external view returns (uint8 action, uint8 score, bytes32 report);
}
```

The call does not revert. Where a signal cannot be established it reports an
unavailability sentinel (`type(uint16).max`, or `type(uint32).max` for
`lpLockRemaining`) rather than a zero that would read as clean.

**Full conformance** adds `ILaunchEscrow` and `ILaunchRemediation`: the venue
holds proceeds, releases them under a 90-day cap, and refunds pro-rata on an
upheld claim.

## Thresholds are calibration, not measurement

The activation thresholds the standard fixes exist so two implementations of the
same vector produce the same number. They are **not** claims about where abuse
begins: no published population establishes that a deployer keeping 30% of
supply is abusive and one keeping 29% is not.

So a deployment using a different threshold **must publish that threshold, and
the population, window and method it was calibrated against**. A threshold
nobody can see is a threshold nobody can check, and a score nobody can recompute
is the thing this standard exists to replace.

## Status

| | |
|---|---|
| Standard | merged into `ethereum/ERCs` |
| Reference implementation | CC0-1.0, with 35 conformance cases, eleven weight profiles and two extension schemas |
| Behind it | 22 contracts and 265 tests covering detector, registry, directory, guard, escrow, remediation and adjudication |
| Live | a service scoring Base launches today, establishing three of the twelve signals and reporting the rest unavailable rather than clean |

The standard requires a directory to be deployed through the
[EIP-7997](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7997.md)
factory where that factory is present, publishing the salt and the init code
hash, so the address follows from the factory, the salt and the init code alone,
and anyone can derive it and confirm the bytecode at it is what it claims to be.
Under `CREATE` the address depends on the deployer's nonce and establishes
neither.

## Design decisions worth knowing

**Price decline is excluded.** Scoring outcome would flag every honest failure
as fraud and would be gameable in reverse, since a deployer who rugs a rising
token would score well. Restricting the vector to deployer actions makes the
score answer "what did this person do", which is the only question a remedy can
rest on.

**The subject is the party that brought the launch, not the signing address.**
Most venues deploy the token themselves from a factory, so the applicant never
appears as a creator on chain. The subject is therefore declared by the venue,
which is also the only reading under which detection tells a buyer anything, since a
signal every launch at a venue shares cannot separate one launch from another.

**A low score is not evidence of legitimacy.** Publishing the signals tells
deployers exactly what is measured. This raises the cost of abuse; it does not
eliminate it.

## Authors

ERC-8376 is co-authored by:

- **Leigh Cronian** ([@cybercentry](https://github.com/Cybercentry)),
  <leigh.cronian@cybercentry.co.uk>. Builds and operates the detector, registry,
  directory and guard
- **Chris Johnson**, <chris@virtuals.io>. Adoption: getting chains, wallets and
  explorers to read the score at the point of purchase

The same two co-authored ERC-8126 and ERC-8196, both Final.

This repository is maintained by Leigh Cronian.

## Licence

[CC0-1.0](LICENSE), a public domain dedication. Copy it, implement it, compete
with it.

## This repository

| Path | What |
|---|---|
| [`assets/ReferenceDetector.sol`](assets/ReferenceDetector.sol) | reference detector, as published with the ERC |
| [`assets/…Conformance_Test_Cases-v0.1.md`](assets/20260830-ERC_8376_Conformance_Test_Cases-v0.1.md) | 35 conformance cases |
| [`assets/…Reference_Weight_Profiles-v0.1.md`](assets/20260830-ERC_8376_Reference_Weight_Profiles-v0.1.md) | eleven weight profiles and their `privilegedPowers` masks |
| [`assets/…Impersonation_Extension_Schema-v0.1.md`](assets/20260830-ERC_8376_Impersonation_Extension_Schema-v0.1.md) | impersonation extension schema |
| [`assets/…Sellability_Extension_Schema-v0.1.md`](assets/20261001-ERC_8376_Sellability_Extension_Schema-v0.1.md) | sellability extension schema |
| [`assets/…Flow_Diagram-v0.1.svg`](assets/20260830-ERC_8376_Token_Launch_Abuse_Detection_and_Remediation_Flow_Diagram-v0.1.svg) | detection, escrow and remediation flow |

The 22 contracts and 265 tests behind the live service are not published here:
what the standard specifies is CC0-1.0, the indexing pipeline and any heuristics
beyond the twelve base signals are not.
