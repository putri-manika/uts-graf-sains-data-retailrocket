# UTS Graf Sains Data: Rekomendasi RetailRocket dengan Neo4j (AuraDB)

## Identitas

-   **Nama:** Putri Manika Rukmamaya
-   **NIM:** 23031554091
-   **Periode P1:** 2015-06-12 s.d. sebelum 2015-07-03
-   **Periode P2:** 2015-07-24 s.d. sebelum 2015-08-14

## Deskripsi

Project ini merupakan pengerjaan UTS mata kuliah **Graf Sains Data**
dengan dataset interaksi e-commerce **RetailRocket**. Data dimodelkan
sebagai graf menggunakan **Neo4j AuraDB**, kemudian digunakan untuk
membangun Vector Space Model (VSM), menghitung similarity antar-user,
melakukan rekomendasi menggunakan KNN, serta mengevaluasi rekomendasi
dengan Hit@5 dan NDCG@5.

> **Catatan:** Graf dijalankan di Neo4j AuraDB. Kredensial koneksi
> dibaca dari file `.env`.

## Struktur Project

``` text
.
├── data/
│   ├── events.csv
│   ├── train_events.csv
│   └── test_events.csv
├── img/
│   ├── sebelum.png
│   └── sesudah.png
├── uts_graf.ipynb
├── requirements.txt
├── .env
└── README.md
```

## Environment dan Dependensi

Versi driver Neo4j yang digunakan:

``` text
neo4j 6.4.1
```

`requirements.txt` menggunakan:

``` text
neo4j>=5.20
```

Penyesuaian ini dilakukan karena kode menggunakan `execute_query`, yang
tidak tersedia pada versi driver Neo4j 4.x.

Library utama yang digunakan:

-   `pandas`
-   `numpy`
-   `scipy`
-   `scikit-learn`
-   `networkx`
-   `matplotlib`
-   `tqdm`
-   `python-dotenv`
-   `neo4j`

## Konfigurasi Neo4j AuraDB

Credential disimpan pada file `.env`:

``` text
NEO4J_URI=...
NEO4J_USER=...
NEO4J_PASSWORD=...
```

Kemudian dibaca melalui Python:

``` python
from dotenv import load_dotenv
import os

load_dotenv(override=True)

URI = os.getenv("NEO4J_URI")
AUTH = (os.getenv("NEO4J_USER"), os.getenv("NEO4J_PASSWORD"))
```

Koneksi diuji dengan:

``` python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(URI, auth=AUTH)
driver.verify_connectivity()
```

## 1. Persiapan dan Filtering Data

Dataset `events.csv` berisi **2.756.101 interaksi** pada periode 3 Mei
sampai 18 September 2015.

Kolom utama:

-   `timestamp`: waktu interaksi dalam Unix epoch milidetik
-   `visitorid`: ID pengunjung
-   `event`: jenis interaksi (`view`, `addtocart`, `transaction`)
-   `itemid`: ID produk
-   `transactionid`: ID transaksi

Timestamp dikonversi menjadi datetime:

``` python
df_raw["dt"] = pd.to_datetime(df_raw["timestamp"], unit="ms")
```

### Periode

**P1 (train):**

``` text
2015-06-12 <= t < 2015-07-03
```

Hasil awal: **416.564 baris**

**P2 (test):**

``` text
2015-07-24 <= t < 2015-08-14
```

Hasil awal: **431.332 baris**

Pada data train, hanya visitor dengan minimal 3 event yang dipertahankan
untuk mengecilkan graf dan mengurangi beban pemrosesan AuraDB.

Hasil akhir train:

-   **199.527 baris**
-   **29.786 visitor**
-   **43.605 item**
-   `VIEWED`: 187.020
-   `ADDED_TO_CART`: 9.102
-   `BOUGHT`: 3.405

Data test tidak difilter dan digunakan untuk evaluasi.

File hasil:

``` text
data/train_events.csv
data/test_events.csv
```

## 2. Pemodelan Graf Neo4j

Skema graf yang digunakan:

``` text
(:Visitor)-[:VIEWED]->(:Item)
(:Visitor)-[:ADDED_TO_CART]->(:Item)
(:Visitor)-[:BOUGHT]->(:Item)
```

Node:

-   `Visitor`
-   `Item`

Relationship:

-   `VIEWED`
-   `ADDED_TO_CART`
-   `BOUGHT`

Constraint unik dibuat untuk `visitorId` dan `itemId`.

### Komposisi Graf

Hasil graf:

  Jenis              Jumlah
  --------------- ---------
  Visitor            29.786
  Item               43.605
  VIEWED            187.020
  ADDED_TO_CART       9.102
  BOUGHT              3.405

Total relationship interaksi adalah **199.527**, sama dengan jumlah
baris data train.

## 3. Vector Space Model (VSM)

Matriks interaksi dibangun dengan:

-   `view` = bobot 1
-   `addtocart` = bobot 3
-   `transaction` = bobot 5

Bobot tersebut digunakan sebagai asumsi bahwa interaksi yang lebih kuat
menunjukkan minat yang lebih besar.

### VSM User

Setiap user direpresentasikan sebagai vektor dengan dimensi sebesar
jumlah item:

``` text
43.605 dimensi
```

### VSM Item

Setiap item direpresentasikan sebagai vektor dengan dimensi sebesar
jumlah user:

``` text
29.786 dimensi
```

Shape matriks utama:

``` text
(29.786, 43.605)
```

Sparsity:

``` text
99,99%
```

NetworkX digunakan untuk memvisualisasikan sampel hubungan user-item dan
ego-network.

## 4. Pemilihan Proyeksi

Dua proyeksi dibandingkan:

-   user-user
-   item-item

Hasil estimasi dari sampel 2.000 simpul:

                                User-User          Item-Item
  -------------------- ------------------ ------------------
  Ukuran penuh            29.786 × 29.786    43.605 × 43.605
  Density                         0,00103            0,00289
  Tetangga rata-rata           sekitar 31        sekitar 126
  Perkiraan nonzero      sekitar 0,9 juta   sekitar 5,5 juta
  Waktu sampel                 0,04 detik         0,05 detik

Proyeksi **user-user** dipilih karena ukurannya lebih kecil, lebih
sparse, dan sesuai dengan pendekatan KNN berbasis user pada tahap
analytics.

## 5. Similarity User

Similarity dihitung menggunakan **cosine similarity** pada VSM user.

User acuan dipilih secara acak dengan:

``` text
seed = 42
```

dengan syarat user memiliki minimal 10 item.

User acuan:

``` text
122944
```

Top-5 similarity:

    Visitor ID   Cosine Similarity
  ------------ -------------------
        271927              0,3333
        242590              0,2981
        243356              0,2887
        684077              0,2236
        746281              0,1443

Hasil similarity kemudian ditambahkan ke graf menggunakan relationship:

``` text
(:Visitor)-[:SIMILAR_TO {score}]->(:Visitor)
```

Cypher untuk melihat 5 similarity tertinggi:

``` cypher
MATCH (a:Visitor {visitorId: $id})-[s:SIMILAR_TO]->(b:Visitor)
RETURN b.visitorId, s.score
ORDER BY s.score DESC
LIMIT 5
```

## 6. KNN Recommendation

KNN menggunakan:

-   `k = 5`
-   metric = cosine
-   representasi = VSM user
-   algoritma = brute force

User dipilih secara acak dari user yang terdapat pada train dan test
menggunakan:

``` text
seed = 42
```

User yang terpilih:

``` text
126107
```

Skor rekomendasi item dihitung dari:

``` text
similarity tetangga × bobot interaksi
```

Item yang sudah pernah dilihat/dilakukan interaksi oleh user target
dikeluarkan dari kandidat rekomendasi.

Top-5 rekomendasi yang diperoleh:

``` text
[93101, 272976, 275406, 91873]
```

Hanya tersedia 4 item baru karena tetangga tidak memiliki lebih banyak
item yang belum pernah berinteraksi dengan user target.

### 3 Item yang Diiklankan

``` text
93101
272976
275406
```

## 7. Evaluasi

Ground truth berasal dari item yang berinteraksi dengan user target pada
periode test.

Untuk user `126107`, terdapat 2 item pada data test:

``` text
39047
426031
```

Kedua item tersebut tidak terdapat pada katalog train sehingga mengalami
**cold-start item**.

Hasil:

``` text
Hit@5 = 0
NDCG@5 = 0
```

### Evaluasi Tambahan

Pada sampel 200 user acak dengan seed 42:

  Metode          Hit@5   NDCG@5
  ------------- ------- --------
  KNN             0,010   0,0041
  Popularitas     0,010   0,0039

Sekitar **67,3%** dari 14.650 interaksi test pada user kandidat memiliki
item yang tersedia di katalog train.

## Kesimpulan

Pendekatan KNN berbasis user belum menunjukkan keunggulan yang jelas
dibandingkan baseline popularitas. Perbedaannya sangat kecil pada sampel
evaluasi.

Beberapa faktor yang memengaruhi hasil:

1.  Data sangat sparse, dengan sparsity sekitar **99,99%**.
2.  Median interaksi user hanya sekitar **3 item**.
3.  Median jumlah user per item hanya **1 user**.
4.  Terdapat masalah **cold-start item** pada periode test.

Beberapa kemungkinan pengembangan:

-   memperpanjang periode data train,
-   menggunakan pembobotan TF-IDF,
-   mencoba pendekatan berbasis item,
-   atau menggunakan metode rekomendasi lain yang lebih sesuai untuk
    data sparse.

## Cara Menjalankan

1.  Siapkan Python environment.
2.  Install dependency dari `requirements.txt`.
3.  Siapkan database Neo4j AuraDB.
4.  Buat file `.env` dan isi credential Neo4j.
5.  Pastikan `data/events.csv` tersedia.
6.  Jalankan notebook `uts_graf.ipynb` dari awal sampai
    akhir.
7.  Pastikan gambar hasil visualisasi tersedia di folder `img/`.
