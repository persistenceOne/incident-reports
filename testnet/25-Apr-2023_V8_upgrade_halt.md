# Testnet V8 Upgrade Chain-Halt Postmortem

**Date of incident**: 25th April, 2023

**Authors**: Max

**Status**: Complete

**Summary**: Chain halt during the Persistence test-core-1 chain upgrade to v8

**Impact**: Downtime for 3h - no impact on data or IBC connections.

**Root Causes**: missing state migration to initialize big.Int value from nil to 0

**Trigger**: Slashing of one validator due to downtime - triggers nil panic

**Resolution**: Update logic to check for nil and also migrate state in EndBlocker (rolling migration)

**Detection**: Shared crash reports in Slack and Discord short after update.

##
## Lessons Learned


### What went wrong

* Missed migration for existing state. We've been testing V8 upgrade internally using empty chain state,
should have tested against existing validator set. But also trigger slashing.


### Where we got lucky

* Slashing happened very soon after testnet upgrade, otherwise debugging could be overnight.
* We were able to migrate state in a rolling migration of EndBlocker, instead of doing anoter release proposal

##
## Timeline

* Apr 25, 2023 at 11:16 PM GMT+2 reported issue in Slack
* Apr 25, 2023 at 02:11 PM GMT+2 released fixed binary

### Fixing commits

https://github.com/persistenceOne/persistence-sdk/pull/397/files
https://github.com/persistenceOne/persistenceCore/pull/195/files


### Example crash log

```
ERR CONSENSUS FAILURE!!! err="runtime error: invalid memory address or nil pointer dereference" module=consensus stack="goroutine 2889 [running]:\nruntime/debug.Stack()\n\truntime/debug/stack.go:24 +0x65\ngithub.com/tendermint/tendermint/consensus.(*State).receiveRoutine.func2()\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:732 +0x4c\npanic({0x28df500, 0x4b56f50})\n\truntime/panic.go:884 +0x212\nmath/big.(*Int).Set(...)\n\tmath/big/int.go:74\ngithub.com/cosmos/cosmos-sdk/types.Dec.Clone(...)\n\tgithub.com/cosmos/cosmos-sdk@v0.46.11/types/decimal.go:222\ngithub.com/cosmos/cosmos-sdk/types.Dec.ImmutOp({0x368db98?}, 0x32c1978, {0x36bd960?})\n\tgithub.com/cosmos/cosmos-sdk@v0.46.11/types/decimal.go:235 +0x56\ngithub.com/cosmos/cosmos-sdk/types.Dec.Quo(...)\n\tgithub.com/cosmos/cosmos-sdk@v0.46.11/types/decimal.go:347\ngithub.com/persistenceOne/persistence-sdk/v2/x/lsnative/staking/keeper.Keeper.Slash({{0x368db98, 0xc0040abf10}, {0x36bd960, 0xc000ee86a0}, {0x36ad9e0, 0xc0013fb900}, {0x36bfbc0, 0xc000b97080}, {0x36be680, 0xc0001264e0}, ...}, ...)\n\tgithub.com/persistenceOne/persistence-sdk/v2@v2.1.0-rc1/x/lsnative/staking/keeper/slash.go:134 +0xcaf\ngithub.com/persistenceOne/persistence-sdk/v2/x/lsnative/slashing/keeper.Keeper.HandleValidatorSignature({{_, _}, {_, _}, {_, _}, {_, _}}, {{0x36aac58, 0xc00012a000}, ...}, ...)\n\tgithub.com/persistenceOne/persistence-sdk/v2@v2.1.0-rc1/x/lsnative/slashing/keeper/infractions.go:87 +0x12c8\ngithub.com/persistenceOne/persistence-sdk/v2/x/lsnative/slashing.BeginBlocker({{0x36aac58, 0xc00012a000}, {0x36be4c0, 0xc025489e00}, {{0xb, 0x0}, {0xc02dab90f0, 0xb}, 0xa8c6dc, {0x369ab7c, ...}, ...}, ...}, ...)\n\tgithub.com/persistenceOne/persistence-sdk/v2@v2.1.0-rc1/x/lsnative/slashing/abci.go:23 +0x23d\ngithub.com/persistenceOne/persistence-sdk/v2/x/lsnative/slashing.AppModule.BeginBlock(...)\n\tgithub.com/persistenceOne/persistence-sdk/v2@v2.1.0-rc1/x/lsnative/slashing/module.go:161\ngithub.com/cosmos/cosmos-sdk/types/module.(*Manager).BeginBlock(_, {{0x36aac58, 0xc00012a000}, {0x36be4c0, 0xc025489e00}, {{0xb, 0x0}, {0xc02dab90f0, 0xb}, 0xa8c6dc, ...}, ...}, ...)\n\tgithub.com/cosmos/cosmos-sdk@v0.46.11/types/module/module.go:484 +0x398\ngithub.com/cosmos/cosmos-sdk/baseapp.(*BaseApp).BeginBlock(_, {{0xc02971ab60, 0x20, 0x20}, {{0xb, 0x0}, {0xc02dab90f0, 0xb}, 0xa8c6dc, {0x369ab7c, ...}, ...}, ...})\n\tgithub.com/cosmos/cosmos-sdk@v0.46.11/baseapp/abci.go:183 +0x843\ngithub.com/tendermint/tendermint/abci/client.(*localClient).BeginBlockSync(_, {{0xc02971ab60, 0x20, 0x20}, {{0xb, 0x0}, {0xc02dab90f0, 0xb}, 0xa8c6dc, {0x369ab7c, ...}, ...}, ...})\n\tgithub.com/tendermint/tendermint@v0.34.27/abci/client/local_client.go:280 +0x118\ngithub.com/tendermint/tendermint/proxy.(*appConnConsensus).BeginBlockSync(_, {{0xc02971ab60, 0x20, 0x20}, {{0xb, 0x0}, {0xc02dab90f0, 0xb}, 0xa8c6dc, {0x369ab7c, ...}, ...}, ...})\n\tgithub.com/tendermint/tendermint@v0.34.27/proxy/app_conn.go:81 +0x55\ngithub.com/tendermint/tendermint/state.execBlockOnProxyApp({0x36abf28?, 0xc005fe5920}, {0x36b5b50, 0xc00f939cc0}, 0xc000b5ad20, {0x36bf348, 0xc00ae63ab8}, 0xa8c6db?)\n\tgithub.com/tendermint/tendermint@v0.34.27/state/execution.go:307 +0x3dd\ngithub.com/tendermint/tendermint/state.(*BlockExecutor).ApplyBlock(_, {{{0xb, 0x0}, {0xc0026845b0, 0x8}}, {0xc0026845c0, 0xb}, 0x1, 0xa8c6db, {{0xc01b91f720, ...}, ...}, ...}, ...)\n\tgithub.com/tendermint/tendermint@v0.34.27/state/execution.go:140 +0x171\ngithub.com/tendermint/tendermint/consensus.(*State).finalizeCommit(0xc001344e00, 0xa8c6dc)\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:1661 +0xafd\ngithub.com/tendermint/tendermint/consensus.(*State).tryFinalizeCommit(0xc001344e00, 0xa8c6dc)\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:1570 +0x2ff\ngithub.com/tendermint/tendermint/consensus.(*State).enterCommit.func1()\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:1505 +0xaa\ngithub.com/tendermint/tendermint/consensus.(*State).enterCommit(0xc001344e00, 0xa8c6dc, 0x0)\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:1543 +0xccf\ngithub.com/tendermint/tendermint/consensus.(*State).addVote(0xc001344e00, 0xc023bf8280, {0xc00487c120, 0x28})\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:2165 +0x18dc\ngithub.com/tendermint/tendermint/consensus.(*State).tryAddVote(0xc001344e00, 0xc023bf8280, {0xc00487c120?, 0xc023bb9f00?})\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:1963 +0x2c\ngithub.com/tendermint/tendermint/consensus.(*State).handleMsg(0xc001344e00, {{0x36862c0?, 0xc01c0f2708?}, {0xc00487c120?, 0x0?}})\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:861 +0x170\ngithub.com/tendermint/tendermint/consensus.(*State).receiveRoutine(0xc001344e00, 0x0)\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:768 +0x3f9\ncreated by github.com/tendermint/tendermint/consensus.(*State).OnStart\n\tgithub.com/tendermint/tendermint@v0.34.27/consensus/state.go:379 +0x12d\n"
``` 
