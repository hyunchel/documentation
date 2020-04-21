---
id: beacon-node
title: Prysm's Beacon Node
sidebar_label: Beacon node
---

The beacon node shipped with Prysm is the keystone component of the Ethereum 2.0 protocol. It is responsibile for running a full [Proof-of-Stake](/docs/glossaries/terminology#proof-of-stake-pos) blockchain, known as a beacon chain, which uses distributed consensus to agree on blocks both [proposed](/docs/glossaries/terminology#propose) and [attested](/docs/glossaries/terminology#attest) on by [validators](/docs/glossaries/terminology#validator) in the network. Beacon nodes communicate their processed blocks to their peers via a P2P \(peer-to-peer\) network, which also manages the lifecycle process of active [validator clients](/how-prysm-works/validator-clients).

## Beacon node functionality

At runtime, the beacon node initialises and maintains a number of services that are all vital to providing all the features of Ethereum 2.0. In no particular order, these services include:

* A [**blockchain** **service**](#blockchain-service) which processes incoming blocks from the network, advances the beacon chain's state, and applies a fork choice rule to select the best head block.
* An [**operations service**](#operations-service) prepares information contained in beacon blocks received from peers \(such as block deposits and attestations\) for inclusion into new validator blocks.
* A [**core package** ](#core-package)containing Ethereum 2.0 core functions, utilities, and state transitions required for conformity with the protocol.
* A [**sync service**](#sync-service) which both queries nodes across the network to ensure latest [canonical head](/docs/glossaries/terminology#canonical-head-block) and state are synced and processes incoming block announcements from peers.
* An [**ETH 1.0 service**](#eth1-service) that listens to latest event logs from the validator deposit contract and the ETH 1.0 blockchain.
* A [**public RPC server**](#public-rpc-server) that requests information about the beacon chain's state, the latest block, validator information, etcetera.
* A [**P2P server**](p2p-networking) which handles the lifecycle of peer connections and facilitates broadcasting across the network.
* A **full test suite** for running simulation on Ethereum 2.0 state transitions, benchmarks and conformity tests across clients.

We isolate each of these services into separate packages, each responsible for its own lifecycle, logging and dependency management. Each Prysm service implements an interface to start, stop, and verify its status at any time.

## Blockchain service

The blockchain service is arguably the most important part of the project, as it allows the network to reach consensus on the state of the protocol itself. It is responsible for handling the lifecycle of blocks, and applies the [fork choice rule](/docs/glossaries/terminology#fork-choice-rule) and [state transition function](/docs/glossaries/terminology#state-transition-function) provided by the [core package](#core-package) to advance the beacon chain.

In Ethereum 2.0, blocks can be proposed in intervals known as _slots_, which are period of seconds. During a slot, proposers are assigned to create and send blocks into the beacon node for acceptance. It is possible, however, that proposer may fail to do their job at their assigned slot; in this case, the blockchain service processes skipped slots appropriately to ensure that the chain does not stall.

## Operations service

The operations service handles important information contained in blocks on the beacon chain, such as voluntary validator exits, [proposals](/docs/glossaries/terminology#propose), [attestations](/docs/glossaries/terminology#attest), slashings and more. The operation is received from the [sync service](#sync-service) via the [P2P network](p2p-networking), or from data the node retrieves locally.

## Core package

The core package implements the Ethereum 2.0 state transition function, as well as the core helpers and utilities. Every function that manages block processing, epoch processing, validator shuffling and finality is defined within this package. It is designed to be a near-identical translation of the official specification. The aim is to keep this package as free of outside code as possible, and it is comprised of mostly pure functions which do not require access to the other services across Prysm to function.

## Sync service

The sync service has two responsibilities: ensuring the local beacon chain is up-to-date with the latest [canonical head](/docs/glossaries/terminology#canonical-head-block) and state as observed by the network, and to listen and respond to requests for new block announcements from peers. The service was designed to be as independent as possible from the rest of the system, and is the main point of interaction for peers over the [P2P network](p2p-networking). Everything in the sync service runs concurrently through a single `Start()` function, which handles several different message requests and responses.

## ETH1 service

The [ETH1](/docs/glossaries/terminology#eth1) service uses the [go-ethereum ethclient](https://github.com/ethereum/go-ethereum/tree/master/ethclient) to connect to a running Ethereum 1.0 node in order to listen for incoming [validator deposit contract](validator-deposit-contract) logs. The [validator clients](validator-clients) include deposit objects inside of their proposed blocks, and the beacon chain state transition function then activates any pending validators from these deposits.

As the beacon node will need to frequently access information and one cannot rely on perfect latency from the [ETH1](/docs/glossaries/terminology#eth1) node, the service also includes the ability to cache received logs and blocks from the Ethereum 1.0 chain.

## Public RPC server

The public RPC server is one of the most critical components of the beacon node. It implements a variety of methods that [validators ](/docs/glossaries/terminology#validator)connected to the node can query and obtain assignments to propose or attest blocks. The API is defined in a [protobuf](https://developers.google.com/protocol-buffers/) formatted file, and any client that implements the client side of these methods can connect via gRPC to the beacon node and begin requesting data from its public endpoints.