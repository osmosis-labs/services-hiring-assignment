# Services Hiring Assignment

## Summary

In this assignment, you are expected to deliver a sidecar query service that is located between Osmosis front end and the chain.

First, you will be introduced to the concept of scalable incentive rewards distribution. Using that knowledge, you are expected to expose two queries (defined further below) for front end clients to call. Note that you are not expected to implement the algorithm itself. The goal is to build a service while assuming that this incentive rewards framework is what already exists on-chain.

Additionally, there are questions related to this assignment. Please fork this repositry and attach your answers directly to the README.

Some of the questions are architectural in-nature, some might require skimming the Osmosis [chain code](https://github.com/osmosis-labs/osmosis), some reading [the linked paper](https://uploads-ssl.webflow.com/5ad71ffeb79acc67c8bcdaba/5ad8d1193a40977462982470_scalable-reward-distribution-paper.pdf), and for others it might be useful to play around with [the Osmosis app](https://app.osmosis.zone/?from=ATOM&to=OSMO).

The expected time for completion is 10-15 hours.

### Deliverables

- Fork this repository and implement a service as a stand-alone binary
- Implement logic for ingesting data from the Osmosis chain. Mock data with JSON where applicable.
- Expose required endpoints
- Test thoroughly and document it
- Treat this as a production service
- Answer below questions and attach answers in the README

## Context

One of the common problems for blockchains is distributing rewards for fractional ownership of assets in an efficient way. For example, consider providing liquidity into a standard liquidity pool. Upon depositing assets into the pool, a user gets back an amount of liquidity provision (LP) shares that is proportional to their ownership in the pool.

As a protocol, Osmosis wants to incentivize users to provide this liquidity through incentive rewards. At regular intervals (i.e. every day), the protocol mints OSMO tokens that get pipelined into pools based on governance-approved proportions.

The problem now becomes determining how to distribute rewards efficiently to each user. Imagine that there are thousands or even millions of users. A naive implementation would iterate over all users and determine the appropriate allocation to each at the time of distribution [1]. 

Instead, we define a framework that allows users to claim rewards on-demand in constant time. Read the "Sources" section at the bottom for more context about this framework. To achieve constant time claiming, the chain defines a per-pool accumulator that represents incentive reward growth per unit of liquidity in the pool. Its value is ever-increasing. At the time of reward distribution, we allocate a per-unit-of-liquidity distribution amount to the pool accumulator.

For each user, whenever they LP into the pool, the system takes a snapshot of the pool accumulator and associates it with the user. Additionally, if the user adds, removes liquidity, or claims rewards, their accumulator snapshot is updated. The system also persists the number of shares users have after the update.

For example, assume that a pool has 900 units of liquidity from various users and 100 units from user A. At the start, each user's individual snapshot is equal to 0. Pool receives 10000 OSMO. Then, the global accumulator becomes 10000 / 1000 = 10 OSMO / unit of liquidity.

Now, user A decides to claim their rewards. They take the current value of the pool accumulator (10) and subtract their snapshot from it. Finally, they multiply the difference by the number of shares that they have (10 - 0 = 10 * 100 = 1000 OSMO). To reflect that they claimed the amount and to avoid being able to claim next time, the user's pool accumulator snapshot is updated to 10.

In order to incentivize users to LP into an Osmosis pool the FE needs to ability to display APR for that and all other available pools [4]. Additionally, it needs to show the users what their claimable rewards are.

We would like to implement a query service that exposes two endpoints[5]:
1. Pool Incentives APR
 Inputs: Pool ID
 Output: APR string
1. Claimable incentive rewards
 Inputs: Pool ID, user address
 Outputs: Incentive rewards denominated in USDC

Note that each pool has a unique ID.
 
For your main assignment, define the service with the above requirements. It has to query the chain to keep the relevant data in sync with the latest state. You have the flexibility to expose chain queries as needed. Define the chain query APIs needed to pull the relevant data. For simplicity, feel free to mock and read these outputs from JSON files in your project.

Expose HTTP endpoints for the front end and other clients to query.

For these queries, feel free to either persist the chain data in-store/RAM from a separate worker or implement them in a pass-through manner by retrieving the data from the chain directly [6].

You will be evaluated on design choices, algorithm selection, design patterns, testability, readability, and security of your code.

This project is expected to take 10 - 15 hours. Review the "Deliverables" section for details. It is acceptable if you don't have a fully functional project by the end of it. What we are looking for is a foundational structure with only some components fully completed, a deep understanding of architectural direction given the requirements, and the ability to justify choices.

Most importantly, we hope that you have a great experience working on this project. Good luck!

## Questions

Note that some questions might be linked to the problem description by number (e.g. "[1]").

[1] Why is the naive implementation not ideal? What is the time complexity? Can you reason about all the trade-offs between the naive and the implemented approach?

[2] We discussed the usage of rewards accumulators in the context of LPing. What are some other blockchain applications you can think of?

[3] The usage of accumulators in blockchains is common. Block heights and time are ever-increasing. Can you browse the SDK modules in [the Osmosis repository](https://github.com/osmosis-labs/osmosis) and list one additional application of the accumulators? How do they differ from reward accumulators?

[4] While this problem is defined in the context of incentive APR only, what are some other components that go into the pool APR in the production Osmosis frontend?

[5] What are the trade-offs between exposing these queries via a separate service as opposed to defining them directly on the node?

[6] Explain your design choice. For each data entry that is required for the service, explain your choice. Why do did you choose it to be persistent, cache only or pass-through? What are the access patterns (read and write) that you are expecting for each?

[7] Any other comments that you would like to add for reviewers?

## Optional Bonus Question (Hard)

[8] So far we have discussed the reward accumulators in the context of standard [constant-function-market-maker](https://en.wikipedia.org/wiki/Constant_function_market_maker) pools where every user provides liquidity in the price range from [0, infinity]. In Osmosis, such pools are balancer and stableswap How would your service data structures and design choices change if we now expanded the scope to also include concentrated liquidity pools that allow providing liquidity in narrow ranges?

Supporting readings:
- [Uniswap Whitepaper](https://uniswap.org/whitepaper-v3.pdf)
- [Concentrated Liquidity README](https://github.com/osmosis-labs/osmosis/blob/main/x/concentrated-liquidity/README.md)

## Acknowledgements For Reviewer (Not Part of Assignment)

- For CFMM pools, all LPs earn the same APR per unit of liquidity. CFMM pool rewards can also be auto-distributed directly into the pool. As a result, it does not actually make perfect sense to use the accumulators for this context. Accumulators are more applicable for concentrated liquidity positions where each position has a different APR stemming from the custom range. However, we define the assignment this way for simplicity since dealing with concentrated liquidity positions is more difficult.

## Sources

Suggestions: first link is an easier and shorter read that is recommended for this assignment.

Read second one only if interested.

- [Scalable Rewards Distribution Paper](https://uploads-ssl.webflow.com/5ad71ffeb79acc67c8bcdaba/5ad8d1193a40977462982470_scalable-reward-distribution-paper.pdf)
- [F1 Fee Distribution Paper]( https://drops.dagstuhl.de/opus/volltexte/2020/11974/pdf/OASIcs-Tokenomics-2019-10.pdf)




