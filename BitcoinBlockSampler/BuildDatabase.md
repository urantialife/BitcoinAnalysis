### Database schema design
- Disk usage prediction criteria
    - Block height: 620000
    - Transactions: 508923998
    - Addresses: 628875885
    - TxIn: 1242812467
    - TxOut: 1351292737

---
#### Native design (a.k.a. Version 1)
- Schema

```sql
CREATE TABLE IF NOT EXISTS Blk (
    height INTEGER PRIMARY KEY,
    blkhash TEXT NOT NULL UNIQUE);

CREATE TABLE IF NOT EXISTS Tx (
    txid TEXT PRIMARY KEY,
    height INTEGER NOT NULL);

CREATE TABLE IF NOT EXISTS TxIn (
    txid TEXT NOT NULL,
    n INTEGER NOT NULL,
    addr TEXT NOT NULL,
    UNIQUE (txid, n, addr));

CREATE TABLE IF NOT EXISTS TxOut (
    txid TEXT NOT NULL,
    n INTEGER NOT NULL,
    addr TEXT NOT NULL,
    UNIQUE (txid, n, addr));
```

- Disk usage prediction
    - Blk: (8 + 64) * 620000 = 44640000
    - Tx: (64 + 8) * 508923998 = 36642527856
    - TxIn: (64 + 8 + 35) * 1242812467 = 132980933969
    - TxOut: (64 + 8 + 35) * 1351292737 = 144588322859
    - Total: 44640000 + 36642527856 + 132980933969 + 144588322859 = 314256424684 (314 GBytes)
        - With indexes: 44640000*2 + 36642527856*2 + 132980933969*3 + 144588322859*3 = 906082106196 (906 GBytes)

---
#### Index design (a.k.a. Version 2)
- Schema

```sql
CREATE TABLE IF NOT EXISTS BlkID (
    id INTEGER PRIMARY KEY,
    blkhash TEXT NOT NULL UNIQUE);

CREATE TABLE IF NOT EXISTS TxID (
    id INTEGER PRIMARY KEY,
    txid TEXT NOT NULL UNIQUE);

CREATE TAblE IF NOT EXISTS AddrID (
    id INTEGER PRIMARY KEY,
    addr TEXT NOT NULL UNIQUE);

CREATE TABLE IF NOT EXISTS TxIn (
    tx INTEGER,
    n INTEGER,
    addr INTEGER,
    UNIQUE (tx, n, addr));

CREATE TABLE IF NOT EXISTS TxOut (
    tx INTEGER,
    n INTEGER,
    addr INTEGER,
    UNIQUE (tx, n, addr));
```

- Disk usage prediction
    - BlkID: (8 + 64) * 620000 = 44640000
    - TxID: (8 + 64) * 508923998 = 36642527856
    - AddrID: (8 + 35) * 628875885 = 27041663055
    - TxIn (8 + 8 + 8) * 1242812467 = 29827499208
    - TxOut: (8 + 8 + 8) * 1351292737 = 32431025688
    - Total: 44640000 + 36642527856 + 27041663055 + 29827499208 + 32431025688 = 125987355807 (125 GBytes)
        - Total without height field in TxID: 44640000 + 36642527856 + 27041663055 + 29827499208 + 32431025688 = 125987355807 (125 GBytes)
        - With indexes: 44640000*2 + 36642527856*2 + 27041663055*2 + 29827499208*3 + 32431025688*3 = 314233236510 (314 GBytes)

##### Child database tables

- Edge table
```sql
    CREATE TABLE Edge AS
        SELECT other.TxIn.addr AS src, other.TxOut.addr AS dst, COUNT(other.TxIn.tx) AS weight
        FROM other.TxIn
        INNER JOIN other.TxOut ON other.TxIn.tx = other.TxOut.tx
        WHERE other.TxIn.addr != 0 AND other.TxOut.addr != 0
        GROUP BY other.TxIn.addr, other.TxOut.addr;
```

---
#### Hierarchical database design (a.k.a. Version 3)

- Level 1: Index Tables (file: index.db)

```sql
CREATE TABLE IF NOT EXISTS BlkID (
    id INTEGER PRIMARY KEY, -- block height
    blkhash TEXT NOT NULL UNIQUE);

CREATE TABLE IF NOT EXISTS TxID (
    id INTEGER PRIMARY KEY,
    txid TEXT NOT NULL UNIQUE);

CREATE TAblE IF NOT EXISTS AddrID (
    id INTEGER PRIMARY KEY,
    addr TEXT NOT NULL UNIQUE);
```

- Level 2: Core Tables (file: core.db)

```sql
CREATE TABLE IF NOT EXISTS BlkTime (
    blk INTEGER PRIMARY KEY,
    unixtime INTEGER NOT NULL);

CREATE TABLE IF NOT EXISTS BlkTx (
    blk INTEGER NOT NULL,
    tx INTEGER NOT NULL,
    UNIQUE (blk, tx));

CREATE TABLE IF NOT EXISTS TxIn (
    tx INTEGER NOT NULL,
    n INTEGER NOT NULL,
    addr INTEGER NOT NULL,
    btc REAL NOT NULL, 
    UNIQUE (tx, n));

CREATE TABLE IF NOT EXISTS TxOut (
    tx INTEGER NOT NULL,
    n INTEGER NOT NULL,
    addr INTEGER NOT NULL,
    btc REAL NOT NULL,
    UNIQUE (tx, n));
```

- Level 3: Util Tables (file: util.db)

```sql
CREATE TABLE IF NOT EXISTS FirstBlk (
    addr INTEGER PRIMARY KEY,
    blk INTEGER NOT NULL);

CREATE TABLE IF NOT EXISTS UTXO (
    tx INTEGER NOT NULL,
    n INTEGER NOT NULL,
    UNIQUE (tx, n));
```