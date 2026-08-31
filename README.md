# Become a Trobium Super Representative (mainnet)

This guide is for operators on **Ubuntu 20.04 or 22.04 LTS** who want to run a **Super Representative (SR)** node on **Trobium mainnet**. An SR is a DPoS witness: the top **21** voted accounts produce blocks every **3 seconds**. A larger standby pool (127) earns standby pay but does not produce.

Product names: **TROB** (coin), **TROBI** (`1 TROB = 1_000_000 TROBI`), **TRB10** / **TRB20**. Addresses start with `T`.

Use the copy-paste config in [`supernode.conf`](supernode.conf). The mainnet binary [`FullNode.jar`](FullNode.jar) is stored with **Git LFS** (install `git-lfs` before clone; see section 3).

```
Server (Ubuntu 20.04/22.04, JDK 8, chrony) → sync as a full node → applyForSR on-chain
  → freeze TROB + vote → restart with --witness
  → top 21 at next maintenance → produce blocks
```

---

## 1. What you are joining

| Property | Mainnet value |
|----------|----------------|
| Consensus | DPoS, **21** Super Representatives |
| Block interval | 3000 ms |
| Solidify threshold | 70% of SRs |
| Standby pool | 127 |
| Maintenance interval | 1_800_000 ms (30 minutes) — votes and SR set update here |
| P2P version | `1` |
| P2P listen port | **16665** (TCP and UDP) |
| HTTP full node | **16667** |
| HTTP solidity | **16668** |
| gRPC | **16669** |
| JSON-RPC (optional) | **50545** / **50555** |
| Explorer | https://trobchain.com |
| Seed / bootstrap peer | `SEED_NODE_IP:16665` (from ops; fill in [`supernode.conf`](supernode.conf)) |
| Genesis SR | `TA69SAhoYrNbjfJ1k6ovREsGEfDNHoDhXs` |
| Genesis `parentHash` | `957dc2d350daecc7bb6a38f3938ebde0a0c1cedafe15f0edae4256a2907449f6` |

The public HTTP host for mainnet is ops-published (`FULLNODE_URL`, port 16667). Point `trobkit` at that URL (or at your own synced node). There is no Ethereum `chainId` / `gasPrice`; contract fee is `fee_limit` in TROBI.

---

## 2. Ubuntu server setup

Target OS: **Ubuntu 20.04 LTS** or **Ubuntu 22.04 LTS** (amd64). Do not use 24.04: OpenJDK 8 is not in the default archives.

Run everything as a user with `sudo`.

### Hardware

| | Full node (sync / RPC) | Super Representative (producing) |
|--|------------------------|----------------------------------|
| CPU | 8+ cores | **32+ cores** |
| RAM | 16 GB+ | **64 GB+** |
| Disk | 3 TB free | High-performance **SSD, 4 TB+** free |
| Network | — | **100+ MB/s** |

The 8-core / 16 GB floor is enough to join and sync. It is not enough to produce reliably.

### Packages, JDK 8, and clock

**JDK 8 only.** JDK 9+ is not supported.

```bash
sudo apt update
sudo apt install -y openjdk-8-jdk chrony ufw curl python3 ca-certificates

# if more than one Java is installed, pick 8
sudo update-alternatives --config java

java -version
# openjdk version "1.8.0_..."

# Ubuntu's JDK 8 home
ls /usr/lib/jvm/java-8-openjdk-amd64
```

Block production is slot-timed. Enable chrony and confirm the clock is NTP-synced:

```bash
sudo systemctl enable --now chrony
timedatectl status
# expect: System clock synchronized: yes
chronyc tracking
```

Set timezone if you want (does not affect consensus; UTC is fine):

```bash
sudo timedatectl set-timezone UTC
```

### File limits

RocksDB opens many files. Create `/etc/security/limits.d/trobium.conf`:

```bash
sudo tee /etc/security/limits.d/trobium.conf >/dev/null <<'EOF'
trobium  soft  nofile  65535
trobium  hard  nofile  65535
EOF
```

### Firewall (ufw)

| Port | Protocol | Role | Public? |
|------|----------|------|---------|
| **16665** | TCP + UDP | P2P | **Yes — required** |
| 16667 | TCP | HTTP Wallet API | No, unless you serve public RPC |
| 16668 | TCP | HTTP solidity (confirmed) | Same |
| 16669 | TCP | gRPC | Same |
| 50545 / 50555 | TCP | JSON-RPC | Same |
| 10001 | TCP | `node.backup` HA | Only between your two SR machines |
| 5555 | TCP | Native event queue | Internal only |

Allow SSH and P2P; leave HTTP/gRPC/JSON-RPC off the public firewall so they stay on localhost:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 16665/tcp comment 'Trobium P2P TCP'
sudo ufw allow 16665/udp comment 'Trobium P2P UDP'
sudo ufw enable
sudo ufw status verbose
```

A producing SR should not expose `walletExtensionApi` or JSON-RPC to the internet. `supernode.conf` keeps the same ports as a full node so a private RPC listener still works; it does not mean those ports should be world-open.

If you later need HTTP from an admin host only:

```bash
sudo ufw allow from ADMIN_HOST to any port 16667 proto tcp
```

### Disk layout and user

```text
/data/trobium/
  FullNode.jar
  supernode.conf          # chmod 600, contains the witness key
  output-directory/       # chain database (created on first start)
  logs/
```

```bash
sudo mkdir -p /data/trobium/{output-directory,logs}
sudo useradd --system --home /data/trobium --shell /usr/sbin/nologin trobium
sudo cp FullNode.jar supernode.conf /data/trobium/
sudo chown -R trobium:trobium /data/trobium
sudo chmod 600 /data/trobium/supernode.conf
```

---

## 3. `FullNode.jar`

This repo stores the mainnet binary with **Git LFS**: [`FullNode.jar`](FullNode.jar) (~135 MB).

On the Ubuntu host, install Git LFS **before** cloning (or run `git lfs pull` after):

```bash
sudo apt update
sudo apt install -y git git-lfs
git lfs install
git clone <THIS_REPO_URL>
cd sr_guide_trobium
ls -lh FullNode.jar
# expect ~135M, not a tiny pointer file
```

Copy it onto the node with [`supernode.conf`](supernode.conf):

```bash
sudo cp FullNode.jar supernode.conf /data/trobium/
sudo chown trobium:trobium /data/trobium/FullNode.jar /data/trobium/supernode.conf
sudo chmod 600 /data/trobium/supernode.conf
```

Do not use a testnet JAR. If you cloned without LFS and `FullNode.jar` is only a few hundred bytes, run `git lfs install && git lfs pull`.

If you must rebuild instead of using the file in this repo:

```bash
sudo apt install -y git
cd mainnet-trobium-blockchian
./gradlew clean build -x test
sudo cp build/libs/FullNode.jar /data/trobium/
sudo chown trobium:trobium /data/trobium/FullNode.jar
```

---

## 4. Edit `supernode.conf`

Change **only** these operator fields. Leave genesis, `node.p2p.version`, committee flags, and ports as shipped.

1. `node.discovery.external.ip = "YOUR_PUBLIC_IP"` — this machine’s public IPv4. Do not use another node’s IP.
2. `seed.node.ip.list`, `node.active`, and `node.passive` — replace `SEED_NODE_IP` with the mainnet seed host from ops (port **16665**). Same value in all three lists.
3. `localwitness` — see sync vs produce below.
4. Optional: `node.backup.members` if you run a hot standby (`BACKUP_PEER_IP`).

### Sync-only (first start)

Leave the key empty so the process cannot produce:

```hocon
localwitness = [
]
```

Start **without** `--witness` until the node has caught the tip.

### Produce (after sync + on-chain apply)

```hocon
localwitness = [
  "YOUR_WITNESS_PRIVATE_KEY"
]
```

The value is the 64-character hex private key of the witness account (no `0x`). Prefer `chmod 600` on the file. The commented `localwitnesskeystore` path in `supernode.conf` is the alternative if you do not want a plaintext key in config.

Never commit a real key. Never put the key in process argv in shared shell history if you can avoid it (`-p` works but leaks into `ps`).

---

## 5. Commands to run on Ubuntu

Heap: **9 GB minimum** (`-Xms9G -Xmx9G`). On a 64 GB SR box, raise heap (for example `-Xms24G -Xmx24G`) and leave RAM for RocksDB page cache.

All commands below assume the layout in section 2 (`/data/trobium`). Run them **on the Ubuntu host**.

### Start — sync only (no `--witness`)

Do this first. Leave `localwitness = []`. Do **not** pass `--witness` until height matches the explorer.

```bash
cd /data/trobium
sudo -u trobium nohup java -Xms9G -Xmx9G \
  -XX:ReservedCodeCacheSize=256m \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
  -XX:MaxDirectMemorySize=1G \
  -XX:+HeapDumpOnOutOfMemoryError \
  -jar /data/trobium/FullNode.jar \
  -c /data/trobium/supernode.conf \
  -d /data/trobium/output-directory \
  >> /data/trobium/logs/start.log 2>&1 &

echo $! | sudo tee /data/trobium/node.pid
```

### Start — Super Representative (`--witness`)

Only after: node is synced, `applyForSR` + votes are on-chain, and `localwitness` holds the witness private key.

```bash
cd /data/trobium
sudo -u trobium nohup java -Xms24G -Xmx24G \
  -XX:ReservedCodeCacheSize=256m \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
  -XX:MaxDirectMemorySize=1G \
  -XX:+HeapDumpOnOutOfMemoryError \
  -jar /data/trobium/FullNode.jar --witness \
  -c /data/trobium/supernode.conf \
  -d /data/trobium/output-directory \
  >> /data/trobium/logs/start.log 2>&1 &

echo $! | sudo tee /data/trobium/node.pid
```

### Follow logs

```bash
tail -f /data/trobium/logs/start.log
# application logs (created after first start):
tail -f /data/trobium/logs/tron.log
ls -lt /data/trobium/logs/
```

| Log line | Meaning |
|----------|---------|
| `Full node running` | Process started |
| `It's not my turn` | `--witness` is on; this 3s slot belongs to another SR (normal until you are scheduled) |
| `Produce block successfully` / `Block is generated successfully` | You produced a block |
| Height stuck far below explorer | Still syncing, or P2P/firewall (check UDP+TCP **16665** and `external.ip`) |

### Stop (nohup)

```bash
# by saved pid
sudo kill "$(cat /data/trobium/node.pid)"

# or find the Java process
pgrep -af 'FullNode.jar'
sudo kill "$(pgrep -f 'FullNode.jar')"
```

Wait until it exits (`pgrep -af FullNode.jar` prints nothing) before starting again. Do not `kill -9` unless it is stuck; RocksDB needs a clean shutdown.

---

## 6. Verify it is correct and running

Run these **on the Ubuntu host**. HTTP is `http://127.0.0.1:16667`. A healthy node answers within a few seconds after `Full node running` appears in the log.

### 0. Ubuntu host

```bash
lsb_release -a
# Ubuntu 20.04 or 22.04

timedatectl status | grep -E 'synchronized|NTP'
sudo ufw status
java -version
# must print 1.8.0_...
```

### 1. Process

```bash
pgrep -af FullNode.jar
# expect: java ... -jar .../FullNode.jar ... -c .../supernode.conf
```

`--witness` must appear in `pgrep` output only after you intend to produce. If it is missing, you are still in sync-only mode.

### 2. Ports listening

```bash
ss -lntup | grep -E '16665|16667|16668|16669'
```

Expect:

- `16665` TCP (P2P)
- `16667` TCP (HTTP full node)
- `16668` TCP (HTTP solidity)
- `16669` TCP (gRPC)

UDP 16665 should also be open (`ss -lnup | grep 16665`). If 16667 is missing, the HTTP API is not up yet — wait and re-check logs.

### 3. HTTP is alive and this is Trobium mainnet

```bash
curl -s -X POST http://127.0.0.1:16667/wallet/getnowblock \
  -H 'Content-Type: application/json' \
  -d '{"visible":true}' | python3 -m json.tool | head
```

You should see a JSON `block_header` (not a connection error). Then confirm P2P identity:

```bash
curl -s http://127.0.0.1:16667/wallet/getnodeinfo | python3 -m json.tool | head -80
```

Check:

| Field | Expected |
|-------|----------|
| `configNodeInfo.p2pVersion` | `"1"` |
| `configNodeInfo.listenPort` | `16665` |
| `currentConnectCount` | **> 0** (peers). `0` means firewall / wrong `external.ip` / wrong seed |
| `block` | `Num:<height>,ID:<id>` — height should climb |

```bash
curl -s http://127.0.0.1:16667/wallet/getnodeinfo | python3 -c '
import json,sys
n=json.load(sys.stdin)
c=n.get("configNodeInfo") or {}
print("p2pVersion", c.get("p2pVersion"))
print("listenPort", c.get("listenPort"))
print("peers", n.get("currentConnectCount"))
print("block", n.get("block"))
print("solidityBlock", n.get("solidityBlock"))
'
```

If `p2pVersion` is not `1`, you started with the wrong config (bundled `config.conf`). Stop, point `-c` at [`supernode.conf`](supernode.conf), delete the DB only if this was a fresh node that never synced mainnet (`rm -rf /data/trobium/output-directory`), and start again.

### 4. Height is increasing (syncing or live)

```bash
for i in 1 2 3; do
  curl -s -X POST http://127.0.0.1:16667/wallet/getnowblock \
    -H 'Content-Type: application/json' \
    -d '{"visible":true}' \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['block_header']['raw_data']['number'])"
  sleep 6
done
```

Numbers should rise (about one block every 3s once fully synced; faster catch-up while syncing). If the same number repeats for minutes and peers are 0, fix P2P.

### 5. Local tip matches the explorer

Compare `block_header.raw_data.number` with the tip on [https://trobchain.com](https://trobchain.com). Sync from genesis takes hours to days.

If you have ops `FULLNODE_URL`:

```bash
echo -n "local  "; curl -s -X POST http://127.0.0.1:16667/wallet/getnowblock \
  -H 'Content-Type: application/json' -d '{"visible":true}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['block_header']['raw_data']['number'])"

echo -n "remote "; curl -s -X POST "$FULLNODE_URL/wallet/getnowblock" \
  -H 'Content-Type: application/json' -d '{"visible":true}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['block_header']['raw_data']['number'])"
```

Within a few blocks of the remote tip means you are caught up. Then you may fill `localwitness` and restart with `--witness`.

### 6. After `--witness`: producing or waiting for a slot

```bash
pgrep -af FullNode.jar | grep -q -- '--witness' && echo 'witness flag: yes' || echo 'witness flag: NO'

grep -E "It's not my turn|Produce block successfully|Block is generated successfully|Full node running" \
  /data/trobium/logs/start.log /data/trobium/logs/tron.log 2>/dev/null | tail
```

- Synced, `--witness` on, **not** yet top 21: `It's not my turn` (or no produce lines). That is expected.
- Top 21 after maintenance: periodic `Produce block successfully`.
- Confirm the account is a witness:

```bash
curl -s -X POST http://127.0.0.1:16667/wallet/listwitnesses \
  -H 'Content-Type: application/json' \
  -d '{"visible":true}' \
  | python3 -c '
import json,sys
w=json.load(sys.stdin).get("witnesses") or []
print("count", len(w))
for x in w:
    print(x.get("address"), "votes", x.get("voteCount"), "url", x.get("url"))
'
```

Your `T…` address must appear. `isJobs: true` (if present) means you are in the active producing set.

### Failure cheatsheet

| Symptom | What to check |
|---------|----------------|
| `curl` connection refused | Process down, or HTTP not bound yet — `pgrep`, `ss -lntup \| grep 16667`, `tail` logs |
| `p2pVersion` ≠ `1` | Wrong config file |
| `currentConnectCount` = 0 | Firewall UDP+TCP 16665, `external.ip`, seed `SEED_NODE_IP:16665` |
| Height frozen | Peers, disk full (`df -h`), `java -version` is 8 |
| `--witness` but never produces | Not top 21 yet; wait for maintenance (~30 min) after votes; `listwitnesses` |

---

## 7. Become a witness on-chain

Running `--witness` does nothing until this account is a witness **and** ranks in the top 21. Applying is a normal signed transaction; you can broadcast it from a laptop via `trobkit` against `FULLNODE_URL` (or your synced node). You do not need the producing server online for this step.

Default apply cost is **1 TROB** (`ACCOUNT_UPGRADE_COST` = `1_000_000` TROBI). Confirm the live value:

```bash
curl -s -X POST "$FULLNODE_URL/wallet/getchainparameters" \
  -H 'Content-Type: application/json' | grep -A1 getAccountUpgradeCost
```

The URL you register must be a valid URL and at most 256 characters.

### trobkit

```bash
npm install trobkit
```

```js
import { Trobkit } from 'trobkit';

const trobkit = new Trobkit({
  fullHost: process.env.FULLNODE_URL, // mainnet HTTP 16667
  privateKey: process.env.TROB_PRIVATE_KEY,
});

const address = trobkit.defaultAddress.base58;
const url = 'https://your-sr.example';

const applyTx = await trobkit.transactionBuilder.applyForSR(address, url);
const signedApply = await trobkit.trob.sign(applyTx);
console.log(await trobkit.trob.sendRawTransaction(signedApply));
```

### HTTP equivalent

`POST /wallet/createwitness` builds an unsigned tx. Sign it (trobkit or your wallet) and `POST /wallet/broadcasttransaction`.

```bash
curl -s -X POST "$FULLNODE_URL/wallet/createwitness" \
  -H 'Content-Type: application/json' \
  -d '{
    "owner_address": "TYourWitnessAddress................",
    "url": "https://your-sr.example",
    "visible": true
  }'
```

### Freeze (voting power) and vote

Votes equal frozen TROB (Freeze V2). Amounts on the wire are integer **TROBI**.

```js
const amountTrobi = 10_000 * 1_000_000; // 10_000 TROB

const freezeTx = await trobkit.transactionBuilder.freezeBalanceV2(
  amountTrobi,
  'BANDWIDTH', // or 'ENERGY'
  address,
);
const signedFreeze = await trobkit.trob.sign(freezeTx);
await trobkit.trob.sendRawTransaction(signedFreeze);

const voteTx = await trobkit.transactionBuilder.vote(
  { [address]: 10_000 }, // vote_count = frozen TROB you want to assign
  address,
);
const signedVote = await trobkit.trob.sign(voteTx);
await trobkit.trob.sendRawTransaction(signedVote);
```

With `trobkit-scripts` (set `TROB_FULL_HOST` to mainnet `FULLNODE_URL`):

```bash
TROB_AMOUNT_TROB=10000 TROB_RESOURCE=BANDWIDTH npm run stake
TROB_VOTES=TYourWitnessAddress................:10000 npm run vote
```

The SR set is recalculated at the **next maintenance** (~30 minutes). Top 21 produce; everyone else stays in the 127 standby.

### Verify

```bash
curl -s -X POST "$FULLNODE_URL/wallet/listwitnesses" \
  -H 'Content-Type: application/json' \
  -d '{"visible":true}'
```

When `listwitnesses` shows your address, fill `localwitness` in [`supernode.conf`](supernode.conf), stop the node, and start again with `--witness` (section 5). Confirm with section 6 step 6.

---

## 8. After you are producing

### Rewards

Default block pay is **0.075 TROB** per block (`75_000` TROBI). Confirm with `/wallet/getchainparameters` (`getWitnessPayPerBlock`). Standby (top 127) has a separate per-block allowance.

Withdraw accumulated witness pay:

```js
const tx = await trobkit.transactionBuilder.withdrawBlockRewards(address);
const signed = await trobkit.trob.sign(tx);
await trobkit.trob.sendRawTransaction(signed);
```

### Governance

Active SRs can create and approve network proposals (`ProposalCreate` / `ProposalApprove` / `ProposalDelete`). HTTP: `/wallet/proposalcreate`, `/wallet/proposalapprove`, `/wallet/proposaldelete`. List: `/wallet/listproposals`. Chain knobs: `/wallet/getchainparameters`.

### Hot standby (`node.backup`)

Two machines, same witness key, same chain data strategy. Same backup `port` (`10001`). Different `priority`. Each `members` list is the **other** machine’s IP, not your own.

Policy: the node that finishes sync first becomes master; if they finish together, higher `priority` wins. Only one process should produce at a time.

Machine A:

```hocon
node.backup {
  port = 10001
  priority = 8
  members = [
    "BACKUP_PEER_IP"
  ]
}
```

Machine B (this host is `BACKUP_PEER_IP`; `members` is Machine A’s host):

```hocon
node.backup {
  port = 10001
  priority = 5
  members = [
    "SR_PRIMARY_IP"
  ]
}
```

Open TCP 10001 only between those two hosts.

### Security

- Witness private key lives only on the producing box (and the HA peer, if any).
- `chmod 600 supernode.conf`. Do not check the filled file into git.
- Do not expose HTTP 16667, gRPC, or JSON-RPC on a producing node.
- Keep JDK 8 and the OS patched; restrict SSH.

---

## 9. Quick checklist

- [ ] Ubuntu **20.04 or 22.04** LTS, `openjdk-8-jdk`, chrony synced, `ufw` allows 16665 tcp+udp, `nofile` 65535, SSD, 32+ cores / 64 GB+ for production
- [ ] Firewall: public TCP+UDP **16665** only
- [ ] Clone with **Git LFS**, then copy [`FullNode.jar`](FullNode.jar) (~135M) to `/data/trobium/`
- [ ] [`supernode.conf`](supernode.conf): `external.ip` and `SEED_NODE_IP` set; genesis / `p2p.version` untouched
- [ ] Start with the **sync** command in section 5 (`pgrep -af FullNode.jar`, ports 16665/16667 up)
- [ ] Section 6: `p2pVersion` is `1`, `currentConnectCount` > 0, height climbing and matching https://trobchain.com
- [ ] `applyForSR` (≈ 1 TROB), Freeze V2, vote for yourself
- [ ] Confirm in `/wallet/listwitnesses`
- [ ] Fill `localwitness`, restart with `--witness` (section 5)
- [ ] Section 6 step 6: `witness flag: yes`, then `It's not my turn` or `Produce block successfully`
