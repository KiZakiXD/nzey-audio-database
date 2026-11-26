# Database Audio NZey — Hasil Pengujian NZey Player & Pelatihan Database N'Zey Assistant

Repo ini adalah database audio milik NZey, isinya kumpulan klip suara hasil pengujian NZey Player sekaligus bahan pelatihan database N'Zey Assistant. Jadi satu database ini dipakai buat dua keperluan yang saling nyambung: nguji gimana NZey Player menangani berbagai karakter suara di dunia nyata, dan ngelatih N'Zey Assistant supaya makin paham konteks suara di sekitar penggunanya.

## Penjelasan detail

NZey Player itu aplikasi pemutar musik yang serius di urusan kualitas audio — ada rantai DSP, equalizer, normalisasi, sampai bypass DSP buat mode bit-perfect. Masalahnya, suara di dunia nyata itu nggak cuma musik. Ada suara harian yang biasa aja, ada suara keras yang muncul tiba-tiba kayak benturan atau barang pecah, ada juga suara manusia yang sifatnya darurat kayak teriakan. Kalau pemutar dan asisten mau peka sama lingkungan, dua-duanya harus diuji dan dilatih pakai contoh suara yang beneran, bukan asumsi.

Makanya database ini dibangun dengan 3 kelas suara:

- casual — suara harian biasa: suasana jalan, keramaian, orang ngobrol, aktivitas normal sehari-hari. Ini kelas pembandingnya, suara yang seharusnya dianggap wajar.
- crash — suara benturan dan kejadian keras: tabrakan, kaca pecah, logam beradu, benda jatuh. Kelas ini yang paling ekstrem dinamikanya, cocok buat nguji ketahanan rantai audio terhadap transient keras yang datang mendadak.
- scream — teriakan manusia: suara yang secara naluriah berarti ada keadaan darurat, dan justru paling sering ketuker sama suara biasa kalau datanya kurang variatif.

Dari sisi pengujian NZey Player, database ini dipakai buat mastiin pemutar tetap stabil dan enak didengar waktu berhadapan sama materi audio yang karakternya beda-beda jauh — dari ambience pelan sampai benturan keras — termasuk nguji normalisasi volume dan respons DSP-nya. Dari sisi N'Zey Assistant, database yang sama dipakai sebagai bahan pelatihan pengenalan suara: asisten dilatih membedakan mana suara normal, mana suara benturan, dan mana teriakan, supaya ke depannya bisa ngasih respons yang tepat sesuai situasi, bukan asal bunyi keras dianggap sama semua.

Total isi database ini 450 klip, masing-masing kelas 150 klip, semuanya seragam formatnya supaya gampang dipakai ulang buat pengujian maupun pelatihan berikutnya.

## Spesifikasi teknis

| Item | Nilai |
|---|---|
| Format | WAV, PCM 16-bit |
| Sample rate | 16.000 Hz |
| Channel | Mono |
| Durasi per klip | Pas 5 detik |
| Normalisasi | Peak −1 dBFS per klip |
| Klip nyaris hening | Dibuang, ambangnya RMS −45 dBFS |

Format 16 kHz mono ini dipilih karena itu format standar input model pengenalan audio (termasuk YAMNet di TensorFlow), jadi database-nya bisa langsung dipakai buat pelatihan tanpa konversi ulang.

## Isi per kelas

| Kelas | Total | Asli | Augmentasi |
|---|---|---|---|
| casual | 150 | 125 | 25 |
| crash | 150 | 82 | 68 |
| scream | 150 | 103 | 47 |
| Total | 450 | 310 | 140 |

File-nya disusun per folder kelas: casual/casual_001.wav, crash/crash_001.wav, scream/scream_001.wav, dan seterusnya.

Sebagian klip adalah hasil augmentasi, dan itu ditandai apa adanya di metadata.csv (kolom augmented, tekniknya ditulis di notes). Tekniknya standar: pitch shift ±2 semitone, tempo 0.9/1.1, gain ±3 dB, dan campuran noise latar yang pelan. Tiap klip augmentasi diturunkan dari klip asli yang beda-beda, dan porsinya selalu di bawah 50% per kelas. Kelas crash paling banyak augmentasinya karena rekaman benturan yang bagus memang paling susah dicari.

Satu hal penting soal pemakaian: klip augmentasi itu turunan klip asli. Kalau database ini dipakai buat pelatihan model dan datanya mau dibagi train/val/test, klip asli dan semua turunannya harus tetap berada di split yang sama (group split berdasarkan klip asal), supaya nggak terjadi kebocoran data yang bikin akurasi kelihatan lebih bagus dari aslinya.

## Sumber klip

Klip asli berasal dari rekaman Freesound yang diakses lewat Openverse API (pratinjau publik), lalu dipotong jadi segmen 5 detik dan diseragamkan formatnya. Judul, pembuat, lisensi CC, dan link sumber tiap klip tercatat di kolom source pada metadata.csv — kalau mau dipakai di luar keperluan belajar dan riset, cek dulu lisensi per file-nya.

Buat kelas crash dan scream, bunyi kuncinya adalah kejadian singkat, jadi segmennya dipotong pakai jendela yang dipusatkan di momen paling keras. Bunyi pentingnya ada di dalam klip, bukan kepotong di tepi.

## metadata.csv

Kolomnya: filename, class, duration_s, sample_rate, channels, source, augmented, notes.
Isi notes adalah posisi segmen (detik ke-berapa dari file sumbernya) dan teknik augmentasi kalau klipnya hasil augmentasi.

## Cara pakai singkat

1. Load WAV-nya sebagai float32 mono 16 kHz dengan rentang −1..1, misalnya pakai tf.audio.decode_wav atau scipy.io.wavfile terus dinormalisasi.
2. Buat kebutuhan pelatihan pengenalan suara, jalankan model YAMNet (https://tfhub.dev/google/yamnet/1) — tiap klip 5 detik menghasilkan embedding 1024-dim per patch 0,96 detik.
3. Rata-ratakan embedding antar-patch (mean pooling) jadi satu vektor per klip, lalu latih head klasifikasi kecil di atasnya, misalnya dense + softmax 3 kelas.
4. Buat kebutuhan pengujian NZey Player, pakai klip per kelas sebagai materi uji putar: cek kestabilan volume, respons normalisasi, dan perilaku DSP di karakter suara yang berbeda-beda.
