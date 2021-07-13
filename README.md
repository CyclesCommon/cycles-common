# Internet Computer Cycles Common Initiative

With the launch of the [Internet Computer] blockchain, the concepts of [Canisters] and [Cycles] are introduced:

- Canisters are programs running on the blockchain with automatically persisted memory state. They communicate with each other by sending messages.
- Cycles are both a unit of computation cost and a native token that can be transferred between canisters.

## Canisters live and breath cycles, and cycles want to be free!

**Free** as in freedom, not in beer.
Computation and storage are resources must be paid for in cycles, but that is not the point here.

To be more precise, **cycles shall be freely transferable**.
Anything less free is doing more harm than good.

## What are the problems?

1. Canisters can send cycles along with an outgoing message, but there is no common interface to *only* send and receive cycles.

The solution is to propose [a standard interface for common cycle operations](https://github.com/CyclesCommon/cycles-common/pull/1), canisters that follow this standard will be able to exchange cycles with less friction.
For example, the ability to withdraw cycles before uninstalling a canister.

2. Cycles can be transferred from a "verified application" subnet to an ordinary "application" subnet, but not in the other direction.

This restriction seems temporarily, but there is also no sign of when or if it ever will be lifted.

3. Canisters risk service disruption due to cycle depletion.

The catch-22 of receiving cycles is that a canister needs some minimum cycles to run the method that receives cycles.
The only way to top up a canister from outside is to use ledger/cycles minting canisters on the NNS.

4. According to the Interface Specification, there is an "unknown" implementation limit of `MAX_CANISTER_BALANCE`.

At the moment this limit seems to be the max of unsigned 64 bit integer.
However, it is not clear when or if this will continue to be the same.
Even 64 bit might become a problem in the future due to token economy.

## How to contribute?

We are actively investigating solutions to the above problems, and we need your contribution.
Please participate in the discussion, make proposals / pull requests, and best of all, implement the cycles interface for your canisters!

Once the first version of the common interface is finalized, we'll start a **Free Your Cycles** movement giving out rewards to canisters that implement this interface.
So stay tuned!

[Internet Computer]: https://internetcomputer.org
[Canisters]: https://sdk.dfinity.org/docs/developers-guide/concepts/canisters-code.html
[Cycles]: https://sdk.dfinity.org/docs/developers-guide/concepts/tokens-cycles.html#how-cycles-work
