# Replication

## [AddressStore](./audit.md#AddressStore)

## MultiSolver

|Field|Type|Description|
|-|-|-|
|as|AddressStore||
|pl|placement.Component||

* *Epoch()*
* *SelfAddr()*
* *ReservationRatio(ctx context.Context, addr Address) (int, error)*
* *SelectRemoteStorages(ctx context.Context, addr Address, excl ...multiaddr.Multiaddr) ([]ObjectLocation, error)*
* *selectNodes(ctx context.Context, addr Address, excl ...multiaddr.Multiaddr) ([]multiaddr.Multiaddr, error)*
* *Actual(ctx context.Context, cid CID) bool*
* *CompareWeight(ctx context.Context, addr Address, node multiaddr.Multiaddr) int*

## ObjectPool

## ReplicationScheduler

## LocalIntegrityVerifier

## ObjectValidator

## PlacementHonorer

## LocationDetector

## StorageValidator

## ObjectReplicator

## Restorer

## ReplicationManager