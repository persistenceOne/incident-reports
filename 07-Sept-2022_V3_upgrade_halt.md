# V3 Upgrade Chain-Halt Postmortem
A copy of this also exists at [google drive](https://docs.google.com/document/d/1zQlPARZfU27Hu-4L1A9BY4OLbCa3fdzeNt5JzbA8rHQ/edit?usp=sharing) 

**Date of incident**: 7th September, 2022.   
**Authors**: Anmol, Puneet.   
**Status**: Complete - action items in progress.   
**Summary**: Chain halt during the Persistence core-1 chain upgrade to v3 and recovery.   
**Impact**: Downtime for 30 hours - no impact on data or IBC connections.   
**Root Causes**: upgrade-info.json was expected but not present during the upgrade; also allowed nodes to start.   
**Trigger**: Doing an incomplete fix in v0.2.3 and then an upgrade for v3.1.1.   
**Resolution**: Provide data to nodes with incorrect apphash and restart network post upgrade.   
**Detection**: Validators discussing on Discord that the chain had stopped after producing 2 blocks. The nodes got an
“appHash mismatch” error and are again trying to get into consensus on a valid appHash.

| Action Item | Type | Owner | Bug |
| --- | --- | --- |----|   
| Apply the smallest fix patches to prod. (ex. default home) | process | Core-1 team | n/a DONE |   
| Bundle everything into code/ binary | prevention | Core-1 team | n/a |   
| Validator upgrade instructions should be zero or basic as required by Cosmos ecosystem | prevention | Core-1 team | n/a |   
| More multinode upgrade e2e tests | prevention | Core-1 team | TODO |   
| Add more logs to chain start and upgrade steps | mitigation | Core-1 team | TODO. |   
| Multi node setup to replicate any mainnet state to be able to resolve issues faster (may or may not be related to upgrade) | mitigation | Core-1 team | TODO |   
| Discuss potential improvements to upgrade handler with Cosmos team | mitigation | Core-1 team | TODO |

## Lessons Learned

### What went well

* Easy communication channels with validators on Discord & Telegram.
* At least 20 validators out of 100 (collectively above 50% voting power) followed the upgrade instructions.
* Eventually able to reproduce the problem during the upgrade.
* Divided the work well between our team members.
* The team tested and was very confident about the recovery process.

### What went wrong

* Proposal time was wrongly estimated in the proposal description.
  Validators were not active during the upgrade (hardly 75%, given that 67% is required for consensus).
  Downtime was ~30 hours.
* No tests or simulations to cover this scenario.
* Issue was caused by human errors following upgrade instructions, which we had no way of confirming.
* It took a long time to reproduce the problem.
* No recovery plan was put in place in case of this error scenario.
* No disaster testing.
* Issue faced in testnet - the solution of “simplifying upgrade instructions/ use cosmovisor with flags” was not the
  best
  solution.
* Did not have sufficient logs for each step of upgrade.
* Some validators that were active had little to no voting power, leading to a lot of time to reach consensus.

### Where we got lucky

* Did not follow the rollback plan proposed by the team, as it would have caused worse issues with the Persistence <>
  Osmosis IBC client being frozen.
* Getting support from contributors such as Alex and Ethan (Confio), Dimi (stake.fish), Marko (Cosmos), Dev (Osmosis),
  Claimens (CryptoCrew), Paranorm (Paranormal Bros), theBro (Cros-nest), toschdev (SG-1), Active Nodes.
* We did not lose any data or transactions, and the bug was recoverable.

## Timeline

**7th Sept:** (// GST is gulf standard time)   
20:20 GST : The Core-1 chain halts for v3 upgrade at height 7791906   
21:09 GST : Most validators are done with the upgrade, waiting for consensus   
Persistence team is taking halt height data backup, starts one foundation node.   
After one foundation started right, all foundations are started up.      
21:25 GST : After consensus is reached, we produce blocks. 7791906, 7791907 are produced.   
21:27 GST : zy | coinhall.org reports error log, consensus is broken, chain is not producing more blocks. We have 67% of
the network active.

```
5:26PM ERR prevote step: ProposalBlock is invalid err="wrong Block.Header.AppHash. Expected
CBE85A119464F9DBBB9FD9B8A5068CC5EDAB812750735044DB23908FB5DCD5D9, got
F8A6FB51AD33D40B1C3C4C99DFD1924FCC12E9C1F33400312DB5389343177DDE" height=7791908 module=consensus round=10
```

^ Referenced as error A
21:27 GST : multiple others report the same error log,
Cros-nest report reverse log

```
Expected F8A6FB51AD33D40B1C3C4C99DFD1924FCC12E9C1F33400312DB5389343177DDE, got
CBE85A119464F9DBBB9FD9B8A5068CC5EDAB812750735044DB23908FB5DCD5D9
```

^ Referenced as error B      
21:48 GST: CryptoCrew says some of their nodes did not panic on the upgrade height.   
21:49 GST : Everyone checks their binary version -> it turns out everyone is on the same one.   
22:00 GST : We decide to wait for 67% of the voting power to be on the right apphash.    
22:30 GST: Consensus is still not up, validators take precaution to not double sign, and everyone checks their go
version.   
22:34 GST : It’s not a go version problem - go is compatible between minor versions.   
22:36 GST : We take a tally and ask validators who got error A vs error B ( -> quite a large divide for only 73% of the
network being live).   
22:50 GST: We think the error is because of the upgrade not working properly for error A type validators - try checking
logs with those validators (logs are exactly the same).   
23:28 GST: anmol writes “The current hypothesis is that, there could have been an issue with some of the nodes with
upgrade-info.json. Chain still upgrades without it. But wasm does not initiallize properly without it. Could be leading
to different app hash.”

**8th Sept:**
00:37 GST: Persistence team is parallely talking to the gpool team to get their node upgraded.   
1:00 GST : Getting difficult to get more voting power - the core team asks the validators to wait for a detailed
rollback plan.   
1:48 GST : [Rollback plan](https://docs.google.com/document/d/1XZ0T6bz2wHDBUhqxBkr52eW4rP9RodxxVaxFXNQiRio/edit#) is
released - it has a risk of double signing if not done correctly.    
1:50 GST: Validators start to stop nodes.   
2:00 GST: Validators are against the rollback plan, and given the active nodes, it would have been impossible to up the
network.    
2:35 GST: We stop the upgrade in regard to risks of double signing and IBCClient update msg part of 7791907.   
2:38 GST: Validators stop node and wait for team instructions for the next day - everyone is tired.    
4:30 GST: Team discusses possible ways to revive the chain and decides to wait.   
7:30 GST: Team is back up.   
The possible solutions are:

1. Rollback plan into detail, given no nodes are active. (possible because ibc-ack was not submitted to osmosis)
2. Export at 7791907   
   a. Start with v0.2.3   
   b. Start with v3.1.1
3. Figure out the bug and just restart

10:30 GST: Discuss the halt with Tushar and the wider team and get debugging.   
11:00 GST: Puneet works with validators to see the exact steps followed during the upgrade (notice a few have not
replaced upgrade-info.json manual instruction) if the exported state is different with both error classes -> it’s the
same, so no fork is required.   
11:00 GST: Anmol starts creating a dev setup to test different scenarios of what could have gone wrong. We have 2
hypotheses:

1. The upgrade did not work for validators
2. Some validators mysteriously skipped the upgrade with 0.2.3

11:00 GST: Sanjeev takes various backups to restart node upgrade height and latest height.   
11:00 GST: Alex Confio joins in to share experience with Tgrade testnet errors and guide direction.   
11:00 GST: CryptoCrew is trying out some more error possibilities.   
15:00 GST: We create a group with Ethan and Alex on Discord to share thoughts.   
15:30 GST: Development testing setup is ready.    
16:00 GST: We test hypothesis 1 - the error is reproduced!   
16:30 GST: Test with different voting power failing and possible revival steps.   
18:00 GST: Replacing data snapshot to fix state and restarting the network works.   
18:30 GST: Test state sync so there are more options.   
18:30 GST: Walk through the revival steps with Dimi, Marko, and Dev.   
19:00 GST: Did not work with state sync, as upgrade height was just 2 blocks behind. Start creating revival doc.   
19:15 GST: All data backups by the team were too large for upload and download, so we used pre-upgrade snapshots
provided by HighStakes validator, synced it to the latest, did a successful upgrade, synced again, and provided that
snapshot.   
20:30 GST:
Post [instructions of revival](https://docs.google.com/document/d/16h43whfILdZQyDm4ksHI_kqFC_4rink273SaQs_Je3k/edit#) on
Discord, notify all validators on DMs and Telegram groups. Follow the steps from the docs and verify node status and do
the instructions accordingly!    
22:00 GST: Start nodes early, as some have already started and prevote is 30%.

**9th Sept:**   
1:20 GST: Reach consensus and chain starts - do communications.   
10:00 GST onwards: Write this document.   
19:30 GST - fin.   
