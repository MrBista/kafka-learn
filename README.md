# Kafka Core Concepts - Lengkap dan Mudah Dipahami

## 1. TOPIC (Topik) 📝

**Analogi Sederhana**: Topic seperti **folder email** atau **channel WhatsApp group**

```
Topic: "user-events"
├── User A registered
├── User B logged in  
├── User C updated profile
└── User D logged out
```

**Karakteristik Topic:**
- Nama topic harus unik dalam cluster
- Bisa menyimpan berbagai jenis message dengan struktur yang sama
- Immutable (sekali ditulis, tidak bisa diubah)
- Append-only (data baru selalu ditambah di akhir)

**Contoh Naming Convention:**
```
e-commerce-order-created
payment-success
user-registration
inventory-updated
notification-email
```

---

## 2. PARTITION (Partisi) 🗂️

**Analogi Sederhana**: Partition seperti **rak buku yang berbeda** dalam satu perpustakaan

```
Topic: "orders"
│
├── Partition 0: [Order1] [Order4] [Order7] ...
├── Partition 1: [Order2] [Order5] [Order8] ...  
└── Partition 2: [Order3] [Order6] [Order9] ...
```

**Mengapa Butuh Partition?**

1. **Parallelism**: Multiple consumer bisa baca secara bersamaan
2. **Scalability**: Data terdistribusi di multiple server
3. **Ordering**: Dalam satu partition, urutan message dijaga

**Contoh Partitioning Strategy:**
```
// Berdasarkan User ID
user_id: 12345 → hash(12345) % 3 = Partition 1
user_id: 67890 → hash(67890) % 3 = Partition 2

// Semua order dari user yang sama akan ke partition yang sama
// Sehingga urutan order per user tetap terjaga
```

---

## 3. PRODUCER (Pengirim Pesan) 📤

**Analogi Sederhana**: Producer seperti **pengirim surat**

**Tanggung Jawab Producer:**
- Memilih topic tujuan
- Menentukan key dan value message
- Memilih partition (atau biarkan Kafka yang tentukan)
- Handle retry jika gagal send

**Contoh Flow Producer:**
```
1. Application Event Occurs
   ├── User registers
   └── Order created
   
2. Create Message
   ├── Key: user-123
   ├── Value: {"userId": 123, "email": "user@example.com"}
   └── Headers: {"source": "user-service"}
   
3. Send to Kafka
   ├── Choose Topic: "user-events"
   ├── Kafka determines Partition based on key
   └── Message stored with offset number
```

**Producer Configuration Penting:**
- `acks`: Level acknowledgment (0, 1, all)
- `retries`: Berapa kali retry jika gagal
- `batch.size`: Ukuran batch untuk efficiency
- `linger.ms`: Waktu tunggu sebelum send batch

---

## 4. CONSUMER (Penerima Pesan) 📥

**Analogi Sederhana**: Consumer seperti **mailman yang ngambil surat**

**Tanggung Jawab Consumer:**
- Subscribe ke satu atau lebih topic
- Baca message secara sequential per partition
- Track offset (posisi terakhir yang sudah dibaca)
- Process message dan commit offset

**Contoh Flow Consumer:**
```
1. Subscribe to Topic
   └── Topic: "user-events"
   
2. Poll for Messages
   ├── Get batch of messages
   ├── Current offset: 1500
   └── Read messages 1500-1520
   
3. Process Messages
   ├── Send welcome email
   ├── Update user analytics
   └── Log activity
   
4. Commit Offset
   └── Mark messages as processed (offset: 1520)
```

---

## 5. CONSUMER GROUP 👥

**Analogi Sederhana**: Consumer Group seperti **tim kantor pos**

```
Topic "orders" dengan 3 Partition:
│
Consumer Group "order-processors":
├── Consumer A → reads from Partition 0
├── Consumer B → reads from Partition 1  
└── Consumer C → reads from Partition 2

Jika Consumer B mati:
├── Consumer A → reads from Partition 0 + 1
└── Consumer C → reads from Partition 2
```

**Keuntungan Consumer Group:**
- **Load Balancing**: Partition didistribusi ke multiple consumer
- **Fault Tolerance**: Jika satu consumer mati, yang lain ambil alih
- **Scalability**: Tambah consumer untuk handle load lebih besar

**Aturan Consumer Group:**
- Satu partition hanya bisa dibaca oleh satu consumer dalam group
- Jika consumer > partition, ada consumer yang idle
- Jika partition > consumer, satu consumer handle multiple partition

---

## 6. OFFSET (Penanda Posisi) 🔢

**Analogi Sederhana**: Offset seperti **bookmark di buku**

```
Partition 0:
[Msg0] [Msg1] [Msg2] [Msg3] [Msg4] [Msg5] ...
   ↑              ↑                    ↑
Offset 0       Offset 2            Offset 5
             (Last Read)        (Latest Message)
```

**Jenis-jenis Offset:**
- **Current Offset**: Posisi terakhir yang sudah di-commit
- **Latest Offset**: Message terbaru di partition
- **Earliest Offset**: Message tertua yang masih tersimpan

**Auto Commit vs Manual Commit:**
```
// Auto Commit (Default)
- Offset otomatis di-commit setiap 5 detik
- Risiko: Message bisa hilang jika crash sebelum commit

// Manual Commit  
- Developer kontrol kapan commit offset
- Lebih aman tapi butuh handling error lebih complex
```

---

## 7. BROKER (Server Kafka) 🖥️

**Analogi Sederhana**: Broker seperti **kantor pos pusat**

**Tanggung Jawab Broker:**
- Menyimpan message dari producer
- Melayani request dari consumer  
- Manage partition dan replication
- Coordinate dengan broker lain

**Kafka Cluster:**
```
Cluster "my-kafka"
├── Broker 1 (Leader untuk Partition 0)
├── Broker 2 (Leader untuk Partition 1)
└── Broker 3 (Leader untuk Partition 2)

Setiap partition punya:
- 1 Leader (handle read/write)
- N Follower (backup data)
```

---

## 8. REPLICATION (Replikasi) 🔄

**Analogi Sederhana**: Replication seperti **fotokopi dokumen penting**

```
Topic "payments" dengan Replication Factor = 3:

Partition 0:
├── Broker 1 (Leader) ✅
├── Broker 2 (Follower) ✅ 
└── Broker 3 (Follower) ✅

Jika Broker 1 mati:
├── Broker 2 (New Leader) ✅
└── Broker 3 (Follower) ✅
```

**In-Sync Replicas (ISR):**
- Replica yang up-to-date dengan leader
- Hanya ISR yang bisa jadi leader baru
- Memastikan data consistency

---

## 9. ZOOKEEPER 🐘

**Analogi Sederhana**: Zookeeper seperti **buku telepon dan koordinator**

**Fungsi Zookeeper:**
- Menyimpan metadata cluster (topic, partition, broker info)
- Leader election untuk partition
- Configuration management
- Service discovery

**Note**: Kafka versi terbaru (2.8+) mulai bisa jalan tanpa Zookeeper dengan KRaft mode.

---

## 10. MESSAGE STRUCTURE 📬

**Struktur Message Kafka:**
```
Message:
├── Key (Optional): "user-123"
├── Value: {"userId": 123, "action": "login"}  
├── Headers: {"source": "auth-service", "version": "v1"}
├── Timestamp: 2024-01-15T10:30:00Z
└── Offset: 1234
```

**Key vs Value:**
- **Key**: Menentukan partitioning, bisa null
- **Value**: Data utama message, bisa null  
- **Headers**: Metadata tambahan untuk routing/filtering

---

## FLOW LENGKAP: Dari Producer ke Consumer

```
1. PRODUCER SIDE:
   App Event → Create Message → Choose Topic → Kafka determines Partition → Store in Log
   
2. KAFKA SIDE:
   Message stored → Replicate to followers → Update ISR → Send ack to producer
   
3. CONSUMER SIDE:
   Poll messages → Process batch → Commit offset → Continue polling
```

---

## PRACTICAL EXAMPLE: E-Commerce Order System

```
Scenario: User membuat order baru

1. ORDER SERVICE (Producer):
   Topic: "orders"
   Key: "user-123" (untuk partitioning)
   Value: {
     "orderId": "ord-456",
     "userId": "user-123", 
     "items": [...],
     "total": 150000
   }

2. KAFKA:
   - Message masuk ke partition berdasarkan hash(user-123)
   - Disimpan dengan offset baru
   - Replicate ke broker lain

3. CONSUMERS:
   ├── INVENTORY SERVICE → Update stock
   ├── PAYMENT SERVICE → Process payment
   ├── EMAIL SERVICE → Send confirmation
   └── ANALYTICS SERVICE → Update metrics
```

Setiap service bisa process secara independent dan pada kecepatan berbeda!

---

## KEY TAKEAWAYS 🎯

1. **Topic** = Channel untuk kategori message
2. **Partition** = Pembagian topic untuk parallelism dan scalability  
3. **Producer** = Pengirim message ke topic
4. **Consumer** = Penerima message dari topic
5. **Consumer Group** = Sekumpulan consumer yang bekerja sama
6. **Offset** = Bookmark posisi pembacaan message
7. **Broker** = Server yang menyimpan dan melayani data
8. **Replication** = Backup data untuk high availability

**Ingat**: Kafka adalah tentang **streaming data** secara **real-time**, **scalable**, dan **fault-tolerant**!