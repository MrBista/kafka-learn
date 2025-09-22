# Kafka CLI Syntax - Penjelasan Detail & Contoh Praktis

## 1. Struktur Dasar Command Kafka

Semua command Kafka mengikuti pattern ini:
```bash
kafka-[tool-name] [required-options] [optional-options]
```

### Format Umum:
```bash
docker exec -it kafka kafka-[command] \
  --bootstrap-server localhost:9092 \
  [other-options]
```

**Penjelasan:**
- `docker exec -it kafka` = Masuk ke container kafka dan jalankan command
- `kafka-[command]` = Tool Kafka yang akan digunakan
- `--bootstrap-server` = Alamat broker Kafka (WAJIB di hampir semua command)
- `localhost:9092` = IP dan port broker Kafka

## 2. KAFKA-TOPICS - Manajemen Topic

### 2.1 Create Topic (Membuat Topic)

#### Syntax Dasar:
```bash
kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic [nama-topic]
```

#### Contoh Sederhana:
```bash
# Membuat topic bernama "test" dengan setting default
docker exec -it kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic test
```

#### Contoh Lengkap dengan Konfigurasi:
```bash
docker exec -it kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --partitions 3 \
  --replication-factor 1 \
  --config retention.ms=604800000
```

**Penjelasan Parameter:**
- `--topic user-events` = Nama topic yang akan dibuat
- `--partitions 3` = Jumlah partition (pembagian data)
- `--replication-factor 1` = Jumlah copy data (1 = no backup, 3 = 2 backup)
- `--config retention.ms=604800000` = Data disimpan 7 hari (dalam millisecond)

### 2.2 List Topics (Melihat Daftar Topic)

```bash
# Melihat semua topic
docker exec -it kafka kafka-topics --list \
  --bootstrap-server localhost:9092
```

**Output akan seperti:**
```
test
user-events
__consumer_offsets
```

### 2.3 Describe Topic (Detail Topic)

```bash
# Melihat detail satu topic
docker exec -it kafka kafka-topics --describe \
  --bootstrap-server localhost:9092 \
  --topic user-events
```

**Output akan seperti:**
```
Topic: user-events      PartitionCount: 3       ReplicationFactor: 1
        Topic: user-events      Partition: 0    Leader: 1       Replicas: 1     Isr: 1
        Topic: user-events      Partition: 1    Leader: 1       Replicas: 1     Isr: 1
        Topic: user-events      Partition: 2    Leader: 1       Replicas: 1     Isr: 1
```

**Penjelasan Output:**
- `PartitionCount: 3` = Topic punya 3 partition
- `Leader: 1` = Broker ID 1 yang handle partition ini
- `Replicas: 1` = Data ada di broker 1
- `Isr: 1` = In-Sync Replicas (broker yang up-to-date)

### 2.4 Delete Topic (Hapus Topic)

```bash
# HATI-HATI: Ini akan hapus topic dan semua datanya!
docker exec -it kafka kafka-topics --delete \
  --bootstrap-server localhost:9092 \
  --topic user-events
```

## 3. KAFKA-CONSOLE-PRODUCER - Mengirim Data

### 3.1 Producer Sederhana

```bash
# Producer basic - ketik message, tekan Enter untuk kirim
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic test
```

**Cara pakai:**
1. Jalankan command di atas
2. Ketik pesan, contoh: `Hello World`
3. Tekan Enter untuk kirim
4. Ketik pesan lain: `This is message 2`
5. Ctrl+C untuk keluar

### 3.2 Producer dengan Key

```bash
# Producer dengan key (format: key:value)
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --property "key.separator=:" \
  --property "parse.key=true"
```

**Cara pakai:**
```
user123:{"action":"login","timestamp":"2024-01-01"}
user456:{"action":"logout","timestamp":"2024-01-01"}
```

**Penjelasan:**
- `key.separator=:` = Pemisah antara key dan value adalah ":"
- `parse.key=true` = Command akan parse key dari input
- `user123` = Key (untuk partitioning)
- `{"action":"login"...}` = Value (data actual)

### 3.3 Producer dengan File Input

```bash
# Buat file dengan pesan
echo -e "message1\nmessage2\nmessage3" > /tmp/messages.txt

# Kirim semua pesan dari file
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic test < /tmp/messages.txt
```

### 3.4 Producer dengan Properties

```bash
# Producer dengan compression dan batching
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --producer-property compression.type=snappy \
  --producer-property batch.size=16384 \
  --producer-property linger.ms=10
```

**Penjelasan Properties:**
- `compression.type=snappy` = Compress data sebelum kirim (hemat bandwidth)
- `batch.size=16384` = Kumpulkan 16KB data sebelum kirim (efficiency)
- `linger.ms=10` = Tunggu 10ms untuk batch lebih banyak message

## 4. KAFKA-CONSOLE-CONSUMER - Membaca Data

### 4.1 Consumer Sederhana

```bash
# Baca dari sekarang (real-time)
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic test
```

**Behavior:**
- Command ini akan "hang" dan tunggu message baru
- Ketika ada message baru masuk, akan tampil di console
- Ctrl+C untuk keluar

### 4.2 Consumer dari Beginning

```bash
# Baca semua data dari awal
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic test \
  --from-beginning
```

**Perbedaan dengan di atas:**
- `--from-beginning` = Mulai baca dari message paling lama
- Akan tampil semua message yang pernah dikirim ke topic

### 4.3 Consumer dengan Consumer Group

```bash
# Consumer dengan group (untuk load balancing)
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group my-app-consumers
```

**Kenapa pakai Consumer Group:**
- Jika ada multiple consumer dengan group sama, mereka akan share load
- Jika topic punya 3 partition, bisa ada 3 consumer parallel
- Kafka track progress per group (consumer offset)

### 4.4 Consumer dengan Key dan Metadata

```bash
# Tampilkan key, value, dan timestamp
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --property print.key=true \
  --property print.timestamp=true \
  --property key.separator=" | " \
  --from-beginning
```

**Output akan seperti:**
```
CreateTime:1640995200000 user123 | {"action":"login","timestamp":"2024-01-01"}
CreateTime:1640995201000 user456 | {"action":"logout","timestamp":"2024-01-01"}
```

**Penjelasan:**
- `CreateTime:1640995200000` = Timestamp message dibuat
- `user123` = Key dari message
- `|` = Separator yang kita set
- `{"action":"login"...}` = Value dari message

### 4.5 Consumer dari Partition dan Offset Tertentu

```bash
# Baca dari partition 0, mulai offset 100
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --partition 0 \
  --offset 100
```

**Penjelasan:**
- `--partition 0` = Hanya baca dari partition 0 (bukan semua partition)
- `--offset 100` = Mulai dari message ke-100 di partition itu

## 5. KAFKA-CONSUMER-GROUPS - Manajemen Consumer Group

### 5.1 List Consumer Groups

```bash
# Lihat semua consumer group
docker exec -it kafka kafka-consumer-groups --list \
  --bootstrap-server localhost:9092
```

**Output:**
```
my-app-consumers
test-group
analytics-consumers
```

### 5.2 Describe Consumer Group

```bash
# Detail consumer group
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group my-app-consumers \
  --describe
```

**Output:**
```
GROUP             TOPIC         PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG   CONSUMER-ID                                     HOST          CLIENT-ID
my-app-consumers  user-events   0          150            155             5     consumer-1-abc123                              /172.17.0.1   consumer-1
my-app-consumers  user-events   1          200            200             0     consumer-2-def456                              /172.17.0.1   consumer-2
my-app-consumers  user-events   2          175            180             5     consumer-3-ghi789                              /172.17.0.1   consumer-3
```

**Penjelasan Kolom:**
- `PARTITION` = Partition yang di-consume
- `CURRENT-OFFSET` = Posisi terakhir yang sudah dibaca consumer
- `LOG-END-OFFSET` = Message terakhir yang ada di partition
- `LAG` = Selisih (seberapa ketinggalan consumer) = LOG-END-OFFSET - CURRENT-OFFSET
- `CONSUMER-ID` = ID unik consumer yang aktif

### 5.3 Reset Consumer Group Offset

```bash
# Reset ke beginning (dry-run dulu)
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group my-app-consumers \
  --topic user-events \
  --reset-offsets \
  --to-earliest \
  --dry-run
```

```bash
# Execute reset (hati-hati!)
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group my-app-consumers \
  --topic user-events \
  --reset-offsets \
  --to-earliest \
  --execute
```

**Options reset offset:**
- `--to-earliest` = Ke message paling awal
- `--to-latest` = Ke message paling baru
- `--to-offset 1000` = Ke offset 1000
- `--to-datetime 2024-01-01T00:00:00.000` = Ke waktu tertentu

## 6. KAFKA-CONFIGS - Konfigurasi

### 6.1 Lihat Konfigurasi Topic

```bash
# Lihat config topic
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name user-events \
  --describe
```

### 6.2 Update Konfigurasi Topic

```bash
# Ubah retention time jadi 24 jam
docker exec -it kafka kafka-configs \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name user-events \
  --alter \
  --add-config retention.ms=86400000
```

**Config penting:**
- `retention.ms` = Berapa lama data disimpan (millisecond)
- `cleanup.policy` = delete (hapus old data) atau compact (keep latest per key)
- `segment.ms` = Ukuran file log segment

## 7. Contoh Skenario Praktis

### Skenario 1: Setup Topic untuk User Events

```bash
# 1. Buat topic
docker exec -it kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --partitions 3 \
  --replication-factor 1

# 2. Cek topic berhasil dibuat
docker exec -it kafka kafka-topics --describe \
  --bootstrap-server localhost:9092 \
  --topic user-events

# 3. Test kirim data
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --property "key.separator=:" \
  --property "parse.key=true"

# 4. Test baca data (buka terminal baru)
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --property print.key=true \
  --property print.timestamp=true \
  --from-beginning
```

### Skenario 2: Monitor Consumer Lag

```bash
# 1. Buat consumer group
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --group monitoring-test \
  --from-beginning

# 2. Cek status consumer group
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group monitoring-test \
  --describe

# 3. Reset offset jika perlu
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group monitoring-test \
  --topic user-events \
  --reset-offsets \
  --to-latest \
  --execute
```

## 8. Tips Debugging Command

### Cek Connection ke Broker

```bash
# Test basic connection
docker exec -it kafka kafka-topics --list \
  --bootstrap-server localhost:9092
```

Jika error: `Error while executing topic command : Connection refused`
- Cek apakah container kafka running: `docker ps`
- Cek logs: `docker logs kafka`

### Cek Topic Existence

```bash
# Cek apakah topic ada
docker exec -it kafka kafka-topics --list \
  --bootstrap-server localhost:9092 | grep "topic-name"
```

### Test Producer/Consumer

```bash
# Terminal 1: Producer
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic test

# Terminal 2: Consumer  
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic test \
  --from-beginning
```

Ketik message di Terminal 1, harus muncul di Terminal 2.

## 9. Command Shortcuts

Buat alias untuk mempermudah:

```bash
# Di ~/.bashrc atau ~/.zshrc
alias kt='docker exec -it kafka kafka-topics --bootstrap-server localhost:9092'
alias kp='docker exec -it kafka kafka-console-producer --bootstrap-server localhost:9092'
alias kc='docker exec -it kafka kafka-console-consumer --bootstrap-server localhost:9092'
alias kg='docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092'

# Contoh usage:
kt --list
kp --topic test
kc --topic test --from-beginning
kg --list
```

## 10. Cheat Sheet Commands

```bash
# TOPICS
kt --create --topic myTopic --partitions 3 --replication-factor 1
kt --list
kt --describe --topic myTopic
kt --delete --topic myTopic

# PRODUCER  
kp --topic myTopic
kp --topic myTopic --property "key.separator=:" --property "parse.key=true"

# CONSUMER
kc --topic myTopic
kc --topic myTopic --from-beginning
kc --topic myTopic --group myGroup
kc --topic myTopic --property print.key=true --property print.timestamp=true

# CONSUMER GROUPS
kg --list
kg --group myGroup --describe
kg --group myGroup --topic myTopic --reset-offsets --to-earliest --execute
```