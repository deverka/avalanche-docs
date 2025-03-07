---
sidebar_label: AVAX Token
---
# AVAX

AVAX is the native utility token of Avalanche. It’s a hard-capped, scarce asset that is used to
pay for fees, secure the platform through staking, and provide a basic unit of account between the
multiple Subnets created on Avalanche. `1 nAVAX` is equal to `0.000000001 AVAX`.

## Utility

AVAX is a capped-supply (up to 720M) resource in the Avalanche ecosystem that's used to power the
network. AVAX is used to secure the ecosystem through staking and for day-to-day operations like
issuing transactions.

AVAX represents the weight that each node has in network decisions. No single actor owns
the Avalanche Network, so each validator in the network is given a proportional weight in the
network's decisions corresponding to the proportion of total stake that they own through proof
of stake (PoS).

Any entity trying to execute a transaction on Avalanche pays a corresponding fee (commonly known as
"gas") to run it on the network. The fees used to execute a transaction on Avalanche is burned,
or permanently removed from circulating supply.

## Tokenomics

A fixed amount of 360M AVAX was minted at genesis, but a small amount of AVAX is constantly minted
as a reward to validators. The protocol rewards validators for good behavior by minting them AVAX
rewards at the end of their staking period. The minting process offsets the AVAX burned by
transactions fees. While AVAX is still far away from its supply cap, it will almost always remain an
inflationary asset.

Avalanche does not take away any portion of a validator's already staked tokens (commonly known as
"slashing") for negligent/malicious staking periods, however this behavior is disincentivized as
validators who attempt to do harm to the network would expend their node's computing resources
for no reward.

<!-- vale off -->

AVAX is minted according to the following formula, where $R_j$ is the total number of tokens at 
year $j$, with $R_1 = 360M$, and $R_l$ representing the last year that the values of
$\gamma,\lambda \in \R$ were changed; $c_j$ is the yet un-minted supply of coins to reach $720M$ at
year $j$ such that $c_j \leq 360M$; $u$ represents a staker, with $u.s_{amount}$ representing the
total amount of stake that $u$ possesses, and $u.s_{time}$ the length of staking for $u$.

AVAX is minted according to the following formula, where $R_j$ is the total number of tokens at:

<!-- markdownlint-disable MD013 -->

$$
R_j = R_l + \sum_{\forall u} \rho(u.s_{amount}, u.s_{time}) \times \frac{c_j}{L} \times \left( \sum_{i=0}^{j}\frac{1}{\left(\gamma + \frac{1}{1 + i^\lambda}\right)^i} \right)
$$

<!-- markdownlint-enable MD013 -->

where

$$
L = \left(\sum_{i=0}^{\infty} \frac{1}{\left(\gamma + \frac{1}{1 + i^\lambda} \right)^i} \right)
$$

At genesis, $c_1 = 360M$. The values of $\gamma$ and $\lambda$ are governable, and if changed,
the function is recomputed with the new value of $c_*$. We have that $\sum_{*}\rho(*) \le 1$.
$\rho(*)$ is a linear function that can be computed as follows ($u.s_{time}$ is measured in weeks,
and $u.s_{amount}$ is measured in AVAX tokens):

$$
\rho(u.s_{amount}, u.s_{time}) = (0.002 \times u.s_{time} + 0.896) \times \frac{u.s_{amount}}{R_j}
$$

If the entire supply of tokens at year $j$ is staked for the maximum amount of staking time (one
year, or 52 weeks), then $\sum_{\forall u}\rho(u.s_{amount}, u.s_{time}) = 1$. If, instead,
every token is staked continuously for the minimal stake duration of two weeks, then
$\sum_{\forall u}\rho(u.s_{amount}, u.s_{time}) = 0.9$. Therefore, staking for the maximum
amount of time incurs an additional 11.11% of tokens minted, incentivizing stakers to stake
for longer periods.

Due to the capped-supply, the above function guarantees that regardless of the governance changes,
AVAX will never exceed a total of $720M$ tokens, or $\lim_{j \to \infty} R(j) = 720M$.

<!-- vale on -->
