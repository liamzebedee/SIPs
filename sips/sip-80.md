---
sip: 80
title: Synthetic Futures 
status: WIP
author: Anton Jurisevic (@zyzek), Jackson Chan (@jacko125), Kain Warwick (@kaiynne)
discussions-to: https://research.synthetix.io/t/sip-80-synthetic-futures/183
created: 2020-08-06
requires (*optional): 79
---

## Simple Summary

This SIP proposes the creation of pooled synthetic futures contracts, with the SNX debt pool acting as counterparty to
each trade.

## Abstract

With the Synthetix debt pool as counterparty, users can trade synthetic futures contracts to gain exposure to a range of assets
without holding the asset. PnL and liquidation calculations are simplified by denominating the margin for each contract in sUSD,
which can be minted and burnt as required, Therefore using Synthetix, users will not be exposed to volatility
in the value of their margin, and they will always be liquidated when their margin is completely exhausted,
with no requirement for a maintenance margin or deleveraging mechanisms. For similar reasons, no separate insurance
fund is necessary under this design.

However, as the counterparty to all orders, the SNX debt pool takes on the risk of any skew in the market.
If the number of long and short contracts is balanced, then a fall in one is compensated by a rise in
the other; but if the system is imbalanced, then new sUSD can be minted at the expense of the debt pool.
To combat this, a perpetual-style funding rate is paid from the heavier to the lighter side of the market,
encouraging a neutral balance.

## Motivation
<!--This is the problem statement. This is the *why* of the SIP. It should clearly explain *why* the current state of the protocol is inadequate.  It is critical that you explain *why* the change is needed, if the SIP proposes changing how something is calculated, you must address *why* the current calculation is innaccurate or wrong. This is not the place to describe how the SIP will address the issue!-->

The current design of Synths does not easily provide traders with a mechanism for leveraged trading or for shorting assets. iSynths are an approximation to a short position but have significant trade-offs in their current implementation. Synthetic futures contracts will enable a much expanded trading experience by enabling both leveraged price exposure and short exposure.

## Specification
<!--The specification should describe the syntax and semantics of any new feature, there are five sections
1. Overview
2. Rationale
3. Technical Specification
4. Test Cases
5. Configurable Values
-->

### Overview
<!--This is a high level overview of *how* the SIP will solve the problem. The overview should clearly describe how the new feature will be implemented.-->

There are a number of high level components required for the implementation of synthetic perpetual futures on Synthetix, they are outlined below:

* [Market and Contract Parameters](#market-and-contract-parameters)
* [Leverage and Margins](#leverage-and-margins)
* [Position Fees](#position-fees)
* [Aggregate Debt Calculation](#aggregate-debt-calculation)
* [Liquidations and Keepers](#liquidations-and-keepers)
* [Continuous funding rate](#skew-funding-rate)
* [Next price fulfillment](#next-price-fulfillment)

Each of these components will be detailed below in the technical specification. Together they enable the system to offer leveraged trading,
while charging a funding rate to reduce market skew and tracking the impact to the debt pool of each futures market.

### Rationale
<!--This is where you explain the reasoning behind how you propose to solve the problem. Why did you propose to implement the change in this way, what were the considerations and trade-offs. The rationale fleshes out what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

Given the complexity of the design of synthetic futures, the rationale and trade-offs are addressed in each component in the technical specification below.

### Technical Specification

#### Market and Contract Parameters

A position, opened on a specific market, may be either long or short.
If it's long, then it gains value as the underlying asset appreciates.
If it's short, it gains value as the underlying asset depreciates.
As all contracts are opened against the pooled counterparty, the price of this exchange is
determined by the spot rate read from an on-chain oracle.

A contract is defined by several basic parameters, noting that a particular account will not be able to open more than
one contract at a time: instead it must modify its existing position.

We will use a superscript to indicate that a value is associated with a particular contract. For example \\(q^c\\)
corresponds to the size of contract \\(c\\). If the superscript is omitted, the symbol is understood to refer to
an arbitrary contract.

| Symbol | Description | Definition | Notes |
| ------ | ----------- | ---------- | ----- |
| \\(q\\) | Contract size | \\(q \ := \ \frac{m_e \ \lambda_e}{p_e} \\) | Measured in units of the base asset. Long positions have \\(q > 0\\), while short positions have \\(q < 0\\). for example a short contract worth 10 BTC will have \\(q = -10\\). The contract size is computed from a user's margin and leverage. See the [margin](#leverage-and-margins) section for a definition of terms used in the definition. |
| \\(p\\) | Base asset spot price | - | We also define \\(p^c_e\\), the spot price when contract \\(c\\) was entered. |
| \\(v\\) | Notional value | \\(v \ := \ q \ p\\) | This is the (signed) dollar value of the base currency units on a contract. Long positions will have positive notional, shorts will have negative notional. In addition to the spot notional value, we also define the entry notional value \\(v_e := q \ p_e = m_e \ \lambda_e\\). |
| \\(r\\) | Profit / loss | \\(r \ := \ v - v_e\\) | The profit in a position is the change in its notional value since entry. Note that due to the sign of the notional value, if price increases long profit rises, while short profit decreases. |

Each market, implemented by a specific smart contract, is differentiated primarily by its base asset and the contracts
open on that market. Additional parameters control the leverage offered on a particular market.

| Symbol | Description | Definition | Notes |
| ------ | ----------- | ---------- | ----- |
| \\(C\\) | The set of all contracts on the market | - | We also have the contracts on the long and short sides, \\(C_L\\) and \\(C_S\\), with \\(C = C_L \cup C_S\\). |
| \\(b\\) | Base asset | - | For example, BTC, ETH, and so on. The price \\(p\\) defined above refers to this asset. |
| \\(Q\\) | Market Size | \\[Q \ := \sum_{c \in C}{\|q^c\|} = Q_L + Q_S\\] \\[Q_L \ := \ \sum_{c \in C_L}{\|q^c\|}\\] \\[Q_S \ := \ \sum_{c \in C_S}{\|q^c\|}\\] | The total size of all outstanding contracts (on a given side of the market). |
| \\(V_{max}\\) | Open interest cap | - |  Orders cannot be opened that would cause the notional value of either side of the market to exceed this limit. We constrain both: \\[Q_L \ p \leq V_{max}\\] \\[Q_S \ p \leq V_{max}\\] The cap will initially be \\(1\,000\,000\\) dollars worth on each side of the market. |
| \\(K\\) | Market skew | \\(K \ := \ \sum_{c \in C}{q^c} \ = \ Q_L - Q_S\\) | The excess contract units on one side or the other. When the skew is positive, longs outweigh shorts; when it is negative, shorts outweigh longs. When \\(K = 0\\), the market is perfectly balanced. |
| \\(\lambda_{max}\\) | Maximum Initial leverage | - | The absolute notional value of a contract must not exceed its initial margin multiplied by the maximum leverage. Initially this will be no greater than 10. |

---

#### Leverage and Margins

When a contract is opened, the account-holder chooses their initial leverage rate and margin, from
which the contract size is computed. As profit is computed against the notional value of a contract, higher leverage
increases the contract's liquidation risk.

| Symbol | Description | Definition | Notes |
| ------ | ----------- | ---------- | ----- |
| \\(\lambda\\) | Leverage | \\(\lambda \ := \ \frac{v}{m}\\) | The sign of \\(\lambda\\) reflects the side of the position: longs have positive \\(\lambda\\), while shorts have negative \\(\lambda\\). We also define \\(\lambda_e := \frac{v_e}{m_e}\\), the selected leverage when the position was entered. We constrain the entry leverage thus: \\[\|\lambda_e\| \leq \lambda_{max}\\] Note that the leverage in a position at a given time may exceed this value as its margin is exhausted. |
| \\(m_e\\) | Initial margin | \\(m_e \ := \frac{v_e}{\lambda_e} \ = \ \frac{q \ p_e}{\lambda_e}\\) | This is the quantity of sUSD the user initially spends to open a contract of \\(q\\) units of the base currency. The remaining \\(\|v_e\| - m_e\\) sUSD to pay for the rest of the position is "borrowed" from SNX holders, and it must be paid back when the position is closed. |
| \\(m\\) | Remaining margin | \\(m \ := \ max(m_e + r + f, 0)\\) | A contract's remaining margin is its initial margin, plus its profit \\(r\\) and funding \\(f\\) (described below). When the remaining margin reaches zero, the position is liquidated, so that it can never take a negative value. |

It is important to note that the granularity and frequency of oracle price updates constrains the maximum leverage
that it's feasible to offer. If the oracle updates the price whenever it moves 1% or more, then any contracts
leveraged at 100x or more will immediately be liquidated by any update.

When a contract is closed, the funds in its margin are settled. After profit and funding are computed, the remaining
margin of \\(m\\) sUSD will be minted into the account that created the contract, while any losses out of the initial
margin (\\(max(m_e - m, 0)\\)), will be minted into the fee pool. 

---

#### Position Fees

Users must pay a fee whenever they increase the skew in the position, proportional with the skew they introduce.
No exchange fee is charged for correcting the skew, nor for closing or reducing the size of a position.

| Symbol | Description | Definition | Notes |
| \\(\phi\\) | Exchange fee rate | - | Account holders will be charged only on skew they introduce into the market when modifying orders. Initially, \\(\phi = 0.3\%\\). |

If the user increases the market skew by \\(k\\) units, they will be charged a fee of \\(\phi \ k \ p_e\\) sUSD from their margin.  
For example, if a user opens an order on the heavier side of the market, then they are charged \\(\phi \ v_e\\).
On the other hand, if they are submitting an order on the lighter side of the market, they will not be charged for that
part of their order that reduces the skew, and only for the part that induces new skew.
That is, the fee is \\(max(0, |q| - |K|) \ p_e \ \phi\\) sUSD.

No fee is charged for closing a position so that funding rate arbitrage is more predictable even as skew changes,
and in particular always profitable when opening a position on the lighter side of the market. See the
[funding rate](#skew-funding-rate) section for further details.

---

#### Liquidations and Keepers

Once a position's remaining margin is exhausted, the position must be closed in a timely fashion,
so that its contribution to the market skew and to the overall debt pool is accounted for as rapidly as possible,
necessary for accuracy in funding rate and minting computations.

As price updates cannot directly trigger the liquidation of insolvent positions, it is necessary 
for keepers to perform this work by executing a public liquidation function.

Yet this is not as simple as checking that a position's current remaining margin is zero,
as it may have transiently been exhausted and then recovered.
Therefore the oracle service that provides prices to the market must retain the full price history,
and in order to liquidate a contract it will be sufficient to prove only that there was a price
in that history that exhausted its remaining margin.

In order to pay for this work, the liquidation point will in fact be slightly above
zero remaining margin, and the difference will go to a liquidation keeper.

| Symbol | Description | Definition | Notes |
| \\(P\\) | Liquidation keeper incentive | - | This is a flat fee that is used to incentivise keeper duties. Initially this will be set to \\(P = 20\\) sUSD. |
| \\(m_{min}\\) | Minimum order size | - | The keeper incentive necessitates that orders are at least as large. We will initially choose \\(m_{min} = 100\\) sUSD, corresponding to 5x leverage at the minimum order size relative to \\(P\\). We will require \\(m_{min} \leq m_e\\). |

A position may be liquidated whenever a price is received that causes:

\\[m \leq P\\]

Then the contract is closed, the incentive is minted into the liquidating keeper's wallet at the time
that the keeper executes the liquidation, with the rest of the contract's initial margin going into the fee pool.

It should be borne in mind that if gas prices are too high to allow liquidations to be profitable
for liquidators, then liquidations will likely not be performed.
In this case the system must fall back on relying on good samaritans until the incentive level can
be raised by SCCP, and the extra incentive of unliquidated positions is ultimately paid out of the debt
pool. Future updates may consider a gas-sensitive liquidation incentive.

The liquidation function should take an array of positions to liquidate;
if any of the provided positions cannot be liquidated, or has already been liquidated, the transaction should not fail, but only liquidate the positions that are eligible.

---

#### Skew Funding Rate

Whenever the market is imbalanced in the sense that there is more open value on one side,
SNX holders take on market risk. At a given skew level, SNX holders take on exposure
equal to \\(K \ (p_2 - p_1)\\) as the price moves from \\(p_1\\) to \\(p_2\\).

Therefore a funding rate is levied, which is designed to incentivise balance in the open interest
on each side of the market.
Contracts on the heavier side of the market will be charged funding, while contracts on the lighter
side will receive funding.
Funding will be computed as a percentage charged over time against each position’s notional value,
and paid into or out of its margin. Hence funding affects each contract's liquidation point.

| Symbol | Description | Definition | Notes |
| \\(W\\) | Proportional skew | \\[W \ := \ \frac{K}{Q}\\] | The skew as a fraction of the total market size. |
| \\(W_{max}\\) | Max funding skew threshold | - | The proportional skew at which the maximum funding rate will be charged (when \\(i = i_{max}\\)). Initially, \\(W_{max} = 100\%\\) | 
| \\(i_{max}\\) | Maximum funding rate | - | A percentage per day. Initially \\(i_{max} = 10\%\\). |
| \\(i\\) | Instantaneous funding rate | \\[i \ := \ clamp(\frac{-W}{W_{max}}, -1, 1) \ i_{max} \\]  | A percentage per day. |

The funding rate can be negative, and has the opposite sign to the skew, as funding flows against capital in the market.
When \\(i\\) is positive, shorts pay longs, while when it is negative, longs pay shorts. When \\(K = i = 0\\),
no funding is paid, as the market is balanced.

Being computed against contract notional value, it is worth being aware that
funding becomes more powerful relative to the margin as the base asset appreciates, and less powerful
as it depreciates.

As the SNX debt pool is the counterparty to every contract, it is either the payer or payee of
funding on every position, but it is always receiving more than it is paying. SNX holders
receive \\(- i \ K\\) in funding: this is a positive value which will be paid into the fee pool.

This funding flow increases directly as the skew increases, and also as the funding rate
increases, which itself increases linearly with the skew (up to \\(W_{max}\\)). As the fee pool
is party to \\(Q\\) in open contracts, its percentage return from funding is
\\(\frac{- i K}{Q} \propto W^2\\), so it grows with the square of the proportional skew.
This provides accelerating compensation as the risk increases.

#### Accrued Funding Calculation

Funding accrues continuously, so any time the skew or base asset price changes, so too does
the funding flow for all open contracts. This may occur many times between the open and close
of each contract. The expense of constantly updating all open contracts is prohibitive, so instead any time
the skew changes, the total accrued funding per base currency unit will be recorded, and the
individual funding flow for each contract computed from this.

The base asset price in fact changes in between these events, but funding calculations will
use the spot rates whenever skew modification occur. If the market is active, any
inaccuracy in funding induced as a result is minor.

If the funding flow per base unit at time \\(t\\) is \\(f(t) = i_t \ p_t\\), then the cumulative
funding over the interval \\([t_i, t_j]\\) is:

\\[\int_{t_i}^{t_j}{f(t) \ dt} = F(t_j) - F(t_i)\\]

If the funding rate and price (hence the funding flow) over an interval is constant, then we also have:

\\[F(t_{k+1}) - F(t_k) \ = \ i_{t_k} \ p_{t_k} \ (t_{k+1} - t_k)\\]

If the skew was updated at a finite sequence of times \\(\\{ t_0 \dots t_n \\} \\),
we only need to recompute funding at each of these times and so we have
\\(n\\) such constant funding-flow intervals. All order-modifying operations
alter the funding rate, and so occur exactly at the boundaries of these periods.

Consider the cumulative funding from the initial time \\(t_0\\) up to the current time
\\(t_n\\), this expands as a telescoping sum, noting that \\(F(t_0) = 0\\):

\\[F(t_n) \ = \ F(t_n) - F(t_0) \ = \ \sum_{k=0}^{n-1}{F(t_{k+1}) - F(t_k)} \ = \ \sum_{k=0}^{n-1}{i_{t_k} \ p_{t_k} \ (t_{k+1} - t_k)} \\]

Furthermore,

\\[F(t_j) - F(t_i) \ = \ \sum_{k=i}^{j-1}{i_{t_k} \ p_{t_k} \ (t_{k+1} - t_k)}\\]

So, if the accumulated funding per base unit is tracked in a time series, appended to
any time the funding flow changes, it is possible to compute in constant time the net funding flow owed
to a contract per base unit over its entire lifetime.

In the implementation, it is unnecessary to track the time at which each datum of the cumulative
funding flow was recorded. For convenience, we will reuse \\(F\\) for the sequence, to be accessed 
by index, rather than as a function of time.

Funding will be settled whenever a contract is closed or modified.

| Symbol | Description | Definition | Notes |
| \\(t_{last}\\) | Skew last modified | - | The timestamp of the last skew-modifying event in seconds. |
| \\(F\\) | Cumulative funding sequence | \\[F_0 \ := \ 0\\] | \\(F_i\\) denotes the i'th entry in the sequence of cumulative funding per base unit. \\(F_n\\) will be taken to be the latest entry. |
| \\(F_{now}\\) | Unrecorded cumulative funding | \\[F_{now} \ := F_n + \ i \ p \ (now - t_{last})\\] | The funding per base unit accumulated up to the current time, including since \\(t_{last}\\). |
| \\(j\\) | Last-modified index | \\[j \leftarrow 0\\] at initialisation. | The index into \\(F\\) corresponding to the event that a contract was opened or modified. |
| \\(f\\) | Accrued contract funding | \\[f^c \ := \ \begin{cases} 0 & \ \text{if opening} \ c \\ \\ \newline q^c \ (F_{now} - F_{j^c}) & \ \text{otherwise} \end{cases}\\] | The sUSD owed as funding by a contract at the current time. It is straightforward to query the accrued funding at any previous time in a similar manner. |
| \\(di_{max}\\) | Maximum funding rate of change | - | This is an allowable funding rate change per unit of time. If a funding rate update would change it more than this, only add at most a delta of \\(di_{max} \ (now - t_{last})\\). Initially, \\(di_{max} = 1.25\%\\) per hour. |

Then any time a contract \\(c\\) is modified, first compute the current funding rate by updating market size and skew, where \\(q'\\) is the contract's updated size after modification:

\\[Q \ \leftarrow \ Q + |q'| - |q|\\]
\\[K \ \leftarrow \ K + q' - q\\]

Then update the accumulated funding sequence:

\\[F_{n+1} \ \leftarrow \ F_{now}\\]

Then settle funding and perform the contract update, including:

\\[t_{last} \ \leftarrow \ now\\]
\\[j^c \ \leftarrow \ \begin{cases} 0 & \ \text{if closing} \ c \\ \\ \newline n + 1 & \ \text{otherwise} \end{cases}\\]

---

#### Aggregate Debt Calculation

Each open contract contributes to the overall system debt of Synthetix.
When a contract is opened, it accounts for a debt quantity exactly equal to the value of its initial margin.
That same value of sUSD is burnt upon the creation of the contract. As the price of the base asset moves, however,
the contract’s remaining margin changes, and so too does its debt contribution. In order to efficiently aggregate all
these, each market keeps track of its overall debt contribution, which is updated whenever contracts are opened or
closed.

The overall market debt is the sum of the remaining margin in all contracts.
The possibility of negative remaining margin will also be neglected in the following computations,
as such contributions can exist only transiently while contracts are awaiting liquidation.
So long as insolvent contracts are liquidated within the 24-hour time lock specified in [SIP 40](sip-40.md), 
the risk of a front-minting attack is minimal.
This will simplify calculations, and for the purposes of aggregated debt, 
the remaining margin will be taken to be \\(m = m_e + r + f\\).

The total debt is computed as follows:

\\[D \ := \ \sum_{c \in C}{m^c}\\]
\\[\ \ = \ K \ (p + F_{now})  + \sum_{c \in C}{m_e^c - v_e^c - q \ F_{j^c}}\\]

Apart from the spot price \\(p\\), nothing in this expression changes except when positions are modified,
Therefore we can keep track of everything with one additional variable to be updated on position modification:
\\(\Delta_e\\), holding the sum on the right hand side. Then upon modification of a contract, this value is updated as follows:

\\[\Delta_e \ \leftarrow \ \Delta_e + \delta_e' - \delta_e\\]

Where \\(\delta_e := m_e - q_e \(p_e + F_j)\\) and \\(\delta_e'\\) is its recomputed value after the contract is modified.

| Symbol | Description | Definition | Notes |
| \\(\Delta_e\\) | Aggregate position entry debt correction | \\[\Delta_e \ := \ \sum_{c \in C}{m_e^c - v_e^c - q_e^c \ F_{j^c}}\\] | - |
| \\(D\\) | Market Debt | \\[D \ := \ max(K \ (p + F_{now}) + \Delta_e, 0)\\] | - |

In this way the aggregate debt is efficiently computable at any time.

---

#### Next Price Fulfillment

As with Synth exchanges, it is possible to detect price updates coming from the oracle and
front-run them for risk free profit. To resolve this, any alteration to a position will be a two-stage process.

1. First the user indicates their intention to alter their position by committing to the chain their intended margin, leverage, and market side.
2. After a price update is received, the order is ready to be committed; a keeper finalises the transaction, additionally updating the global funding rate, skew, and debt values. The user's contract is then active; the entry price is established: funding and PnL are computed relative to this time.

The finalisation transaction will be paid for by the user from their ether gas tank (see [SIP-79](sip-79.md)), so that
the keeper who executes it does not bear any cost for doing so.

At each step the user's wallet must contain enough sUSD to service the intended margin after fees, or else the order
will revert or be dropped. This process applies to opening and closing positions, as well as to changing sides or
modifying a contract's overall size, but it will not apply to margin top-ups or withdrawals, as these operations
have no affect on market skew or funding rates. 

Note that in order to ensure exchange fees are predictable, they will be computed relative to
the market conditions at the time the order was submitted, rather than at the confirmation time.
A user may change their committed trade before it is finalised, which will recompute the fees and position size, but if
they do so, this will entail waiting for another price to be received. 

In order to avoid collisions, for the first implementation, there will be only one privileged keeper, whose profits
will be paid into the fee pool. This will be done with a view to upgrading to a more robust and decentralised
transaction relayer framework as the product matures.

---

### Extensions

* Paying a portion of the skew funding rate owed to the fee pool to the lighter side of the market, which would enhance the profitability of taking the counter position on market.
* Make funding rate sensitive to leverage - right now a market with \\(100 \times 10\\) long and \\(500 \times 2\\) open interest is considered balanced, even though the long exposure is much riskier. Some remedies could include:
    * Funding rate that accounts for leverage risk
    * Funding rate as automatic position size scaling, which would automatically bring the market into balance.
* Mechanisms to constrain the overall size of each market, other than leverage and available sUSD.

---

### Smart Contract Interface

TBD

### Test Cases
<!--Test cases for an implementation are mandatory for SIPs but can be included with the implementation..-->
Test cases for an implementation are mandatory for SIPs but can be included with the implementation.

### Configurable Values (Via SCCP)
<!--Please list all values configurable via SCCP under this implementation.-->
Please list all values configurable via SCCP under this implementation.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
