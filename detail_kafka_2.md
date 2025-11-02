# Kafka CLI & Konsep Lengkap - Panduan Komprehensif

## 📋 DAFTAR ISTILAH WAJIB KAFKA

### Core Concepts
- **Broker** = Server Kafka yang menyimpan dan melayani data
- **Topic** = Kategori/channel untuk streaming data (seperti table di database)
- **Partition** = Pembagian topic untuk parallelism & scalability
- **Offset** = ID unik sequential untuk setiap message dalam partition
- **Producer** = Aplikasi yang mengirim data ke Kafka
- **Consumer** = Aplikasi yang membaca data dari Kafka
- **Consumer Group** = Kumpulan consumer yang bekerja sama membaca topic

### Message Structure
- **Key** = Identifier untuk message (untuk partitioning & compaction)
- **Value** = Data actual dari message (payload)
- **Timestamp** = Waktu message dibuat/masuk ke Kafka
- **Headers** = Metadata tambahan (key-value pairs)

### Reliability & Performance
- **Replication Factor** = Jumlah copy data untuk high availability
- **ISR (In-Sync Replica)** = Replica yang up-to-date dengan leader
- **Leader** = Partition replica yang handle read/write
- **Follower** = Partition replica yang sync dari leader
- **ACK (Acknowledgment)** = Konfirmasi bahwa message berhasil diterima
- **Idempotence** = Jaminan message tidak duplikat meski retry
- **Transaction** = Menulis ke multiple topics secara atomic

### Consumer Concepts
- **Consumer Group** = Load balancing & fault tolerance mechanism
- **Rebalancing** = Redistribusi partition ketika consumer join/leave
- **Lag** = Jumlah message yang belum dibaca consumer
- **Offset Commit** = Menyimpan posisi terakhir yang sudah diproses
- **Retention** = Berapa lama data disimpan di Kafka

### Advanced
- **Compaction** = Hanya simpan value terakhir per key
- **Segment** = File storage unit di disk
- **Controller** = Broker yang manage cluster metadata
- **ZooKeeper/KRaft** = Coordination service untuk Kafka cluster

---

## 🔥 PRODUCER ACK (ACKNOWLEDGMENT) - KONSEP PENTING!

ACK menentukan **kapan producer menganggap message berhasil terkirim**. Ini trade-off antara **performance vs durability**.

### ACK Levels

#### **acks=0** (Fire and Forget - TIDAK AMAN!)
```bash
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --producer-property acks=0
```

**Behavior:**
- Producer TIDAK menunggu response dari broker
- Producer kirim message dan langsung lanjut
- **FASTEST** tapi **TIDAK ADA JAMINAN** message sampai

**Kapan Pakai:**
- Data yang boleh hilang (metrics, logs yang tidak kritis)
- Butuh throughput maksimal
- Contoh: IoT sensor data yang kirim setiap detik

**Risiko:**
- Message bisa hilang jika broker crash
- Tidak ada error notification

---

#### **acks=1** (Leader Acknowledgment - DEFAULT)
```bash
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --producer-property acks=1
```

**Behavior:**
- Producer tunggu sampai **LEADER partition** tulis ke disk
- Tidak tunggu follower replica
- **BALANCED** performance & durability

**Kapan Pakai:**
- Most common use case (default setting)
- Balance antara speed & safety
- Contoh: User activity logs, general application events

**Risiko:**
- Jika leader crash SEBELUM follower sync, message bisa hilang
- Masih ada window untuk data loss

**Ilustrasi:**
```
Producer → Leader (ACK ✓) → Follower (sync later)
           ↓
    ACK langsung return
```

---

#### **acks=all / acks=-1** (All In-Sync Replicas - PALING AMAN!)
```bash
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --producer-property acks=all \
  --producer-property min.insync.replicas=2
```

**Behavior:**
- Producer tunggu sampai **LEADER + SEMUA ISR** tulis ke disk
- **SLOWEST** tapi **PALING AMAN**
- Kombinasi dengan `min.insync.replicas` untuk guarantee durability

**Kapan Pakai:**
- Data CRITICAL yang tidak boleh hilang
- Financial transactions, payment events, orders
- Compliance data, audit logs

**Risiko:**
- Lebih lambat (higher latency)
- Jika ISR < min.insync.replicas, producer akan error

**Ilustrasi:**
```
Producer → Leader (write) → Follower 1 (write) → Follower 2 (write)
                                                    ↓
                                            ACK setelah SEMUA selesai
```

---

### Min In-Sync Replicas (Wajib Paham!)

```bash
# Set di topic level
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name critical-payments \
  --alter \
  --add-config min.insync.replicas=2
```

**Penjelasan:**
- `min.insync.replicas=2` = Minimal 2 replica (leader + 1 follower) harus sync
- Jika ISR < min.insync.replicas, producer dengan `acks=all` akan **error**
- Kombinasi WAJIB untuk high durability

**Best Practice:**
```
Replication Factor = 3
Min In-Sync Replicas = 2
ACKs = all

→ Bisa tahan 1 broker failure tanpa data loss
```

---

### Producer Idempotence (Anti-Duplicate)

```bash
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --producer-property enable.idempotence=true
```

**Kenapa Penting:**
- Jika producer retry karena timeout, message bisa duplikat
- `enable.idempotence=true` = Kafka detect & reject duplicate

**Automatically Sets:**
- `acks=all`
- `max.in.flight.requests.per.connection=5`
- `retries=Integer.MAX_VALUE`

**Kapan Pakai:**
- Default for production (hindari duplicate)
- Especially untuk critical events

---

## 👥 CONSUMER GROUP - DEEP DIVE

Consumer Group adalah **mekanisme load balancing & fault tolerance** di Kafka.

### Konsep Dasar

```
Topic: user-events (3 partitions)

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Partition 0 │  │ Partition 1 │  │ Partition 2 │
└─────────────┘  └─────────────┘  └─────────────┘
       ↓                ↓                ↓
┌─────────────────────────────────────────────────┐
│         Consumer Group: "app-consumers"         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │Consumer 1│  │Consumer 2│  │Consumer 3│     │
│  └──────────┘  └──────────┘  └──────────┘     │
└─────────────────────────────────────────────────┘
```

**Aturan Utama:**
1. **Satu partition hanya bisa dibaca oleh SATU consumer dalam group**
2. **Satu consumer bisa baca MULTIPLE partitions**
3. Jika consumer > partitions, ada consumer yang idle
4. Jika consumer < partitions, ada consumer yang baca multiple partitions

---

### Skenario Consumer Group

#### Skenario 1: Perfect Balance
```
3 Partitions + 3 Consumers = Setiap consumer dapat 1 partition
→ OPTIMAL untuk throughput
```

```bash
# Terminal 1
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property client.id=consumer-1

# Terminal 2  
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property client.id=consumer-2

# Terminal 3
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property client.id=consumer-3
```

---

#### Skenario 2: Consumer < Partitions
```
3 Partitions + 1 Consumer = Consumer baca SEMUA partitions
→ BOTTLENECK, throughput rendah
```

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers
```

---

#### Skenario 3: Consumer > Partitions
```
3 Partitions + 5 Consumers = 2 consumers IDLE
→ WASTEFUL, consumer menganggur
```

**Best Practice:**
- Number of consumers ≤ Number of partitions
- Untuk scaling, tambah partition dulu!

---

### Rebalancing (Penting!)

**Rebalancing** = Redistribusi partition assignment ketika:
1. Consumer baru join group
2. Consumer crash/leave group
3. Topic partition bertambah
4. Consumer timeout (heartbeat missed)

```bash
# Lihat rebalancing in action
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group app-consumers \
  --describe
```

**Impact:**
- **STOP-THE-WORLD**: Semua consumer stop processing
- Bisa memakan waktu beberapa detik
- Message processing delay

**Minimize Rebalancing:**
```bash
# Increase session timeout
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property session.timeout.ms=30000 \
  --consumer-property heartbeat.interval.ms=3000
```

---

### Offset Commit Strategies

#### Auto Commit (Default - RISIKO!)
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property enable.auto.commit=true \
  --consumer-property auto.commit.interval.ms=5000
```

**Behavior:**
- Kafka auto-commit offset setiap 5 detik
- **RISIKO**: Jika consumer crash setelah read tapi sebelum process, message hilang

**Ilustrasi:**
```
1. Consumer read message (offset 100)
2. Auto-commit (offset 101) ← COMMIT SUDAH JALAN
3. Consumer crash saat processing
4. Restart → Mulai dari 101 (message 100 HILANG)
```

---

#### Manual Commit (RECOMMENDED!)
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property enable.auto.commit=false
```

**In Code (Java Example):**
```java
// Read message
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

for (ConsumerRecord<String, String> record : records) {
    // Process message
    processMessage(record);
    
    // Commit AFTER processing
    consumer.commitSync();
}
```

**Best Practice:**
- Disable auto-commit
- Commit manually AFTER processing berhasil
- At-least-once delivery guarantee

---

### Consumer Lag Monitoring (CRITICAL!)

```bash
# Check lag per partition
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group app-consumers \
  --describe
```

**Output:**
```
PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
0          1000           1500            500  ← Consumer ketinggalan 500 messages
1          2000           2000            0    ← Up-to-date
2          1500           2500            1000 ← Consumer ketinggalan 1000 messages
```

**LAG Analysis:**
- `LAG = 0` → Consumer up-to-date ✓
- `LAG > 0 dan naik` → Consumer tidak cukup cepat! Scaling needed!
- `LAG > 0 dan turun` → Consumer catching up ✓

**Action Items:**
1. LAG tinggi → Add more consumers
2. LAG stabil → Cek apakah consumer crash/hang
3. LAG spike → Check for slow processing logic

---

## 🎯 PRODUCER CONFIGURATIONS PENTING

### Batching & Compression
```bash
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --producer-property compression.type=snappy \
  --producer-property batch.size=16384 \
  --producer-property linger.ms=10 \
  --producer-property buffer.memory=33554432
```

**Penjelasan:**
- `compression.type=snappy` = Compress sebelum kirim (30-40% saving)
  - Options: `none`, `gzip`, `snappy`, `lz4`, `zstd`
  - `snappy` = balance speed & compression
  - `gzip` = compression terbaik, slow
  - `lz4` = fastest, compression moderate
  
- `batch.size=16384` = Batch 16KB messages sebelum kirim
  - Larger batch = better throughput, higher latency
  
- `linger.ms=10` = Tunggu 10ms untuk batch lebih banyak
  - Trade-off: latency vs throughput
  
- `buffer.memory=33554432` = 32MB buffer untuk pending messages

**Impact:**
```
Without Batching: 100 messages = 100 network requests
With Batching: 100 messages = 5-10 network requests
→ 10x better throughput!
```

---

### Retry & Timeout
```bash
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --producer-property retries=3 \
  --producer-property retry.backoff.ms=100 \
  --producer-property request.timeout.ms=30000 \
  --producer-property delivery.timeout.ms=120000
```

**Penjelasan:**
- `retries=3` = Retry 3x jika gagal
- `retry.backoff.ms=100` = Tunggu 100ms antara retry
- `request.timeout.ms=30000` = Timeout per request
- `delivery.timeout.ms=120000` = Total timeout including retries

---

## 🎯 CONSUMER CONFIGURATIONS PENTING

### Polling & Processing
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property max.poll.records=500 \
  --consumer-property max.poll.interval.ms=300000 \
  --consumer-property fetch.min.bytes=1024 \
  --consumer-property fetch.max.wait.ms=500
```

**Penjelasan:**
- `max.poll.records=500` = Max messages per poll() call
  - Smaller = lower latency, more CPU
  - Larger = higher throughput
  
- `max.poll.interval.ms=300000` = Max time between poll() calls
  - Jika exceed, consumer dianggap dead → rebalancing!
  
- `fetch.min.bytes=1024` = Tunggu minimal 1KB data sebelum return
- `fetch.max.wait.ms=500` = Tunggu max 500ms untuk accumulate data

---

### Offset Management
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group app-consumers \
  --consumer-property auto.offset.reset=earliest \
  --consumer-property enable.auto.commit=false
```

**auto.offset.reset Options:**
- `earliest` = Start dari message paling awal (like --from-beginning)
- `latest` = Start dari message baru (default)
- `none` = Throw exception jika tidak ada offset

**Kapan Digunakan:**
```
earliest → New consumer group, ingest historical data
latest   → New consumer group, only new messages
none     → Strict validation, must have committed offset
```

---

## 📊 TOPIC CONFIGURATIONS PENTING

### Retention Policy
```bash
# Time-based retention (7 hari)
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name user-events \
  --alter \
  --add-config retention.ms=604800000

# Size-based retention (1GB per partition)
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name user-events \
  --alter \
  --add-config retention.bytes=1073741824
```

**Kombinasi:**
- Kafka akan delete berdasarkan **mana yang tercapai duluan**
- Time OR Size limit

---

### Cleanup Policy
```bash
# Delete old data (default)
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name user-events \
  --alter \
  --add-config cleanup.policy=delete

# Compact: Keep latest value per key
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name user-profiles \
  --alter \
  --add-config cleanup.policy=compact
```

**Use Cases:**
- `delete` → Time-series data, logs, events
- `compact` → Database changelogs, user profiles (CDC pattern)

**Compaction Illustration:**
```
Before Compaction:
user1: {name: "John", age: 25}
user1: {name: "John", age: 26}
user1: {name: "John", age: 27}

After Compaction:
user1: {name: "John", age: 27}  ← Only latest kept
```

---

## 🚀 PRODUCTION BEST PRACTICES

### 1. Topic Creation
```bash
docker exec -it kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic critical-orders \
  --partitions 6 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=2592000000 \
  --config compression.type=producer
```

**Rationale:**
- `partitions=6` → Parallelism (adjust based on load)
- `replication-factor=3` → High availability
- `min.insync.replicas=2` → Durability with acks=all
- `retention.ms=30 days` → Keep data 1 bulan
- `compression.type=producer` → Use producer compression

---

### 2. Producer for Critical Data
```bash
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic critical-orders \
  --producer-property acks=all \
  --producer-property enable.idempotence=true \
  --producer-property compression.type=lz4 \
  --producer-property max.in.flight.requests.per.connection=5 \
  --producer-property retries=2147483647
```

---

### 3. Consumer for Critical Processing
```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic critical-orders \
  --group order-processors \
  --consumer-property enable.auto.commit=false \
  --consumer-property max.poll.records=100 \
  --consumer-property max.poll.interval.ms=300000 \
  --consumer-property session.timeout.ms=30000 \
  --consumer-property auto.offset.reset=earliest
```

---

## 📝 MONITORING & TROUBLESHOOTING

### Check Topic Health
```bash
# Describe topic dengan detail
docker exec -it kafka kafka-topics --describe \
  --bootstrap-server localhost:9092 \
  --topic user-events

# Check under-replicated partitions (DANGER!)
docker exec -it kafka kafka-topics --describe \
  --bootstrap-server localhost:9092 \
  --under-replicated-partitions
```

**Red Flags:**
- `Isr` < `Replicas` → Some replicas offline/lagging
- `Leader: -1` → No leader! Partition unavailable!

---

### Check Consumer Health
```bash
# Overall group status
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group app-consumers \
  --describe

# Check all groups
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --list

# Check consumer group state
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group app-consumers \
  --describe \
  --state
```

**States:**
- `Stable` → Normal operation
- `PreparingRebalance` → Rebalancing about to start
- `CompletingRebalance` → Rebalancing in progress
- `Empty` → No active consumers
- `Dead` → Group cleanup in progress

---

### Performance Testing
```bash
# Producer performance test
docker exec -it kafka kafka-producer-perf-test \
  --topic test-perf \
  --num-records 100000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092 acks=1

# Consumer performance test
docker exec -it kafka kafka-consumer-perf-test \
  --bootstrap-server localhost:9092 \
  --topic test-perf \
  --messages 100000 \
  --threads 1
```

---

## 🎓 CHEAT SHEET LENGKAP

```bash
# === TOPICS ===
# Create with production config
kt --create --topic T --partitions 3 --replication-factor 3 \
   --config min.insync.replicas=2 --config retention.ms=604800000

# Check health
kt --describe --topic T
kt --describe --under-replicated-partitions

# === PRODUCER ===
# Critical data (acks=all, idempotent)
kp --topic T \
   --producer-property acks=all \
   --producer-property enable.idempotence=true

# High throughput (batching + compression)
kp --topic T \
   --producer-property compression.type=lz4 \
   --producer-property batch.size=32768 \
   --producer-property linger.ms=10

# === CONSUMER ===
# Safe consumption (manual commit)
kc --topic T --group G \
   --consumer-property enable.auto.commit=false \
   --consumer-property max.poll.records=100

# Reprocess from beginning
kc --topic T --group G \
   --consumer-property auto.offset.reset=earliest

# === CONSUMER GROUPS ===
# Monitor lag (CRITICAL!)
kg --group G --describe

# Reset to beginning (DANGER!)
kg --group G --topic T \
   --reset-offsets --to-earliest --execute

# Reset to specific offset
kg --group G --topic T:0 \
   --reset-offsets --to-offset 1000 --execute

# Reset to timestamp
kg --group G --topic T \
   --reset-offsets --to-datetime 2024-01-01T00:00:00.000 --execute

# === CONFIGS ===
# View topic config
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name T --describe

# Update retention
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name T --alter \
  --add-config retention.ms=86400000
```

---

## ✅ RANGKUMAN KEY TAKEAWAYS

1. **ACKs = Durability Trade-off**
   - `acks=0` → Fast, unsafe
   - `acks=1` → Balanced (default)
   - `acks=all` → Slow, safest (+ min.insync.replicas)

2. **Consumer Groups = Load Balancing**
   - 1 partition = 1 consumer (in same group)
   - Consumers ≤ Partitions (optimal)
   - Monitor LAG constantly!

3. **Idempotence = Anti-Duplicate**
   - Always enable for production
   - Auto-sets acks=all + retries

4. **Manual Commit > Auto Commit**
   - Auto-commit = risk of message loss
   - Manual commit = at-least-once guarantee

5. **Replication = High Availability**
   - replication-factor=3 (standard)
   - min.insync.replicas=2 (with acks=all)
   - Can survive (N-1) broker failures

6. **Monitoring is CRITICAL**
   - Check consumer lag regularly
   - Watch for under-replicated partitions
   - Monitor rebalancing frequency

---

**Semoga dokumentasi ini membantu! 🚀**