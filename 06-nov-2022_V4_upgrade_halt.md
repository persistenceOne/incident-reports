# V4 Upgrade Chain-Halt Postmortem
A copy of this also exists at [google drive](https://docs.google.com/document/d/1c5xtuK7_r2VKJX1SvZCYnwqHpGprTmO9Q4BazszlL6o/edit#heading=h.tbxvxljw7tfw) 
##

**Date of incident**: 6th November, 2022 between 12.39UTC (Block #8647535) and 17.17UTC  
**Authors**: Anmol, Puneet, Jeroen   
**Status**: Resolved with outstanding action items.   
**Summary**: Chain halted approximately 4 days after the Persistence core-1 chain upgrade to v4 due to node non-determinism because of various go-versions.   
**Impact**: 
1. Downtime for ~4 hours and 38 minutes.   
2. The pstake_delegation_ICA channel has closed (no revival possible).
3. 5 validators got tombstoned (=jailed forever after a double signing infraction):  in the process of restarting the node due to not backing up priv_validator_state.json


**Root Causes**: Validators with Go version 1.18 had different apphash from validators with Go version 1.19.   
**Trigger**: The 3rd Staking Epoch of the Epoch module, which was the first time the sort function resulted in different outcomes between versions 1.18 and 1.19. 
**Resolution**: Restarted the network from a snapshot with binary built with go 1.19.3   
**Detection**: Validators noticed ‘appHash mismatch’ and started the discussion on the validator channel on the discord server.



| Action Item | Type | Owner | Bug |
| --- | --- | --- |----|   
| Go version sync across all validator nodes | Prevention | Core-1 team | DONE |   
| Re-establish ICA Channel (Persistence - CosmosHub) | Recovery | pSTAKE Team | DONE |   
| Revert Tombstone Action | Recovery | Core-1 team | DONE |   
| Monitoring script to assess the versions for all nodes on the Core-1 chain before upgrades | Prevention | Core-1 team | TODO |   


##
## Lessons Learned

### What went well

* Easy communication channels with validators on Discord & Telegram.
* Validators were responsive and helped in figuring out the bug and propose possible solutions
* Team was quickly able to get to the root cause of the issue
* Divided the work well between our team members
* Minimum down time of 5 hours on non-determinism error


### What went wrong

* Minor issue with go version upgrade from 1.18 to 1.19, which is ideally should not have caused the issue
* Code not tested against multiple versions of go
* Validators did not perform the best practices of recovering from snapshots
* Side effect of the error lead to closing of ICA channels for liquid staking module
* Tombstoning event for validators who participated in the emergency upgrade, resulting in slashing of the delegators

### Where we got lucky

* We only found 2 app hash mismatch, restricting the possible errors that needed to be explored    
* We did not lose any data or transactions, and the bug was recoverable   


##
## Timeline

6th November 2022:   
16:42 GST: `hyunggi | DSRV` reported app hash mismatch on discord, multiple validators started reporting the same issue and the chain halted   
16:48 GST: Team started looking into the issue   
17:00 GST: Confirmed that validators were seeing 2 kinds of app hashes   


```
ERR prevote step: ProposalBlock is invalid err="wrong Block.Header.AppHash. Expected 2385090C23C6FEE14BA26C42194F7811587D9FFBF3449A759185119757959B0D, got 7B34DB75B3EE35ABB98421BC1214A3527E507611C3C1162B54BD51914423614F

ERR prevote step: ProposalBlock is invalid err="wrong 

Block.Header.AppHash. Expected 7B34DB75B3EE35ABB98421BC1214A3527E507611C3C1162B54BD51914423614F, got 2385090C23C6FEE14BA26C42194F7811587D9FFBF3449A759185119757959B0D
```

17:21 GST: Team started asking validators to share logs for the 2 app hashes, triggered from the staking epoch of lscosmos module.   
17:26 GST: Team started hypothesising difference between the app hash could be due to different go version for the 2 sets of validators with different app hashes.   
17:30 GST: Validators confirmed the 2 sets of apphash for 2 versions   

```
go 1.18
7B34DB75B3EE35ABB98421BC1214A3527E507611C3C1162B54BD51914423614F

go 1.19
2385090C23C6FEE14BA26C42194F7811587D9FFBF3449A759185119757959B0D
```

17:35 GST: Team asked all validators to stop the network officially at height 8647536   
17:40 GST: Team started to reproduce the issue locally to confirm the error.   
19:02 GST: Team made an announcement with detailed steps on restarting the chain from snapshots (3 various types), and build the binary again with go 1.19, published statically linked binaries to the release for v4.0.0.   
19:03 GST: Validators started to restart the nodes with the [given instructions](https://discord.com/channels/796174129077813248/825820268231655425/1038830837631832126)   
19:24 GST: Team stopped foundations nodes to slowdown rounds for the halt height block   
19:49 GST: Team started to keep an eye on all validator rounds to get to the 67%   
21:18 GST: Blocks started and the chain was back up again   
21:26 GST: 5 validators were reported to be tombstoned   

```
evidence:
- '@type': /cosmos.evidence.v1beta1.Equivocation
  consensus_address: persistencevalcons1dmjc55ve2pe537hu8h8rjrjhp4r536g5jlnlk8
  height: "8647536"
  power: "1914227"
  time: "2022-11-06T12:39:19.628225786Z"
- '@type': /cosmos.evidence.v1beta1.Equivocation
  consensus_address: persistencevalcons1gnevun33uphh9cwkyzau5mcf0fxvuw6cyrf29g
  height: "8647536"
  power: "1046837"
  time: "2022-11-06T12:39:19.628225786Z"
- '@type': /cosmos.evidence.v1beta1.Equivocation
  consensus_address: persistencevalcons1ak5f5ywzmersz4z7e3nsqkem4uvf5jyya62w3c
  height: "8647536"
  power: "1578191"
  time: "2022-11-06T12:39:19.628225786Z"
  - '@type': /cosmos.evidence.v1beta1.Equivocation
  consensus_address: persistencevalcons15fxjrujvsc0le9udjf63504sd4lndcam8ep4cs
  height: "8647536"
  power: "522127"
  time: "2022-11-06T12:39:19.628225786Z"
- '@type': /cosmos.evidence.v1beta1.Equivocation
  consensus_address: persistencevalcons1m83jqu6q6aqcshnq0yjrdra9nj8rgz79mndh3j
  height: "8647536"
  power: "166180"
  time: "2022-11-06T12:39:19.628225786Z"
```

21:28 GST: Team acknowledges the tombstoned validators and asked for more details   
21:30 GST: Reason for tombstoned was validators that used the snapshots that did not preserve the `priv_validator_state.json`.   
21:31 GST: Some validators proposed un-tombstoning of validators based on Secret Network and ChihuahuaChain   
21:48 GST: Team confirmed and asked to wait for the next day for more information on next steps   
21:50 GST: Team confirms that the ICA channel has been closed and will affect the lscosmos module   



