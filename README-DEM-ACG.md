# Miningcore - DEM & ACG Pool Setup

This branch adds support for **Deutsche eMark (DEM)** and **Aurum Crypto Gold (ACG)** mining pools.

## Changes from upstream

### `src/Miningcore/Blockchain/Bitcoin/BitcoinJobManagerBase.cs`
- Added fallback for legacy daemons that report `netmhashps` instead of `NetworkHashps`
- Fix: `NetworkHashrate = NetworkHashps > 0 ? NetworkHashps : NetMHashps * 2000000`

### `src/Miningcore/Blockchain/Bitcoin/DaemonResponses/GetMiningInfoResponse.cs`
- Added `NetMHashps` property with `[JsonProperty("netmhashps")]` for legacy daemon support

## Why these changes?
The Deutsche eMark daemon is a legacy daemon that reports network hashrate as `netmhashps` 
(in MH/s) instead of the modern `NetworkHashps` (in H/s). Without this fix, the pool 
dashboard shows 0 for network hashrate.

The multiplier `* 2000000` converts MH/s to H/s correctly for DEM.

## Requirements
- Ubuntu 22.04 / 24.04
- Docker & Docker Compose
- Deutsche eMark daemon (vfvalidierung/deutsche_emark:2.0.1)
- Aurum Crypto Gold daemon (abadom3030/aurum-node:latest)

## Setup

### 1. Clone this repo
```bash
git clone https://github.com/Darth999/miningcore.git
cd miningcore
git checkout dem-acg-pool
```

### 2. Configure
Copy the example config and adjust to your setup:
```bash
cp examples/dem-acg-pool.json config.json
```

Edit `config.json` and replace all `YOUR_*` placeholders:
- `YOUR_POSTGRES_PASSWORD` – PostgreSQL password
- `YOUR_POOL_NAME` – Your pool name (shown in coinbase)
- `YOUR_DEM_WALLET_ADDRESS` – Your DEM payout wallet
- `YOUR_DEM_PUBKEY` – Your DEM public key
- `YOUR_ACG_WALLET_ADDRESS` – Your ACG payout wallet
- `YOUR_RPC_USER` / `YOUR_RPC_PASSWORD` – Daemon RPC credentials

### 3. Build & Run
```bash
docker build -t miningcore:latest .
docker compose up -d
```

## Miner Configuration

### Deutsche eMark (DEM)
- **Stratum URL:** `stratum+tcp://YOUR-SERVER-IP:9990`
- **Algorithm:** SHA256d
- **Tested with:** Bitaxe Gamma 601/602, NerdQaxe+++

### Aurum Crypto Gold (ACG)
- **Stratum URL:** `stratum+tcp://YOUR-SERVER-IP:9989`
- **Algorithm:** SHA256d

## Notes
- DEM requires `"hasLegacyDaemon": true` in the pool config
- Bitaxe firmware 2.13.1: `stratum_task: Failed to process mining notification` errors are harmless – mining works correctly
- Network hashrate graph updates hourly, allow 2-3 hours for the graph to populate

## Based on
- [djspacedevil/miningcore](https://github.com/djspacedevil/miningcore)
- Original: [oliverw/miningcore](https://github.com/oliverw/miningcore)

## ACG Daemon Setup (Important!)

### 1. Enable txindex
Add to your `aurum.conf`:
Then restart the node with `-reindex` once:
```bash
# In docker-compose.yml temporarily add:
command: aurumd -datadir=/root/.bitcoin -reindex
# After reindex remove the -reindex flag
```

### 2. Import Pool Private Key
The ACG daemon needs the pool's private key to verify block rewards:
```bash
aurum-cli importprivkey YOUR_ACG_PRIVATE_KEY "" true
```
Without this, blocks will be classified as orphaned with "missing tx details" error.

### 3. hasLegacyDaemon
ACG requires `"hasLegacyDaemon": true` in the pool paymentProcessing config.
