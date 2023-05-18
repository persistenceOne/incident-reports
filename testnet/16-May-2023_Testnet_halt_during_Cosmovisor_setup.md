# Postmortem: Testnet halt during Cosmovisor setup on persistence testnet validators

**Date of incident**: 16th May, 2023

**Authors**: Manasi

**Status:** Complete

**Summary**: Chain halt while setting up cosmovisor on persistence testnet validators

**Impact**: Downtime for 1h -

1) could not recover Nirvana Validator keys

2) no impact on data or IBC connections

**Root Causes**: Missed to remove cleanup step in an older ansible script to setup cosmovisor

**Trigger**: NA

**Resolution**: Keep all the ansible scripts related to Persistence up to date (github)

**Detection**: Shared incident detail on Discord short after facing the incident.

## Lessons Learned

### What went well

* The team could recover 3 validators with latest node snapshot and backup keys during the recovery process.

### What went wrong

* Downtime was ~1 hours which was not expected for just a tool update.
* Persistence team was running 4 testnet validators which had a large percentage of voting power since not all testnet validators were online and able to sync in the previous testnet chain upgrade (v8.0.0-rc4). If voting power was spread out properly, chain halt would also not have happened.

* Due to unexpected scenario which was caused by human errors, no prior communication was made beforehand to notify other validators

* It took a long time to search for backed up keys.

* No disaster testing.

* Testnet halt could be completely avoided if tool update would have done one by one on each validator. This would also avoid an impact of the human error on total downtime.
