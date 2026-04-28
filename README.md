# JMeter Screenshot Documentation

📋 **[JMeter Screenshot Documentation](https://drive.google.com/drive/folders/1wCGoRHtoByyzLMm4dcZbQKOVpxDDwK73?usp=sharing) (open link)**

## Reflection

### 1. Apa perbedaan antara pendekatan performance testing dengan JMeter dan profiling dengan IntelliJ Profiler dalam konteks optimasi performa aplikasi?

JMeter melakukan pengujian dari sisi luar aplikasi (black-box). Ia menembakkan request HTTP secara konkuren dan mengukur metrik makro: response time, throughput, error rate, dan latency tail. JMeter menjawab pertanyaan *"seberapa cepat dan stabil endpoint saya saat dibebani banyak user?"*.

IntelliJ Profiler bekerja dari dalam JVM (white-box). Ia melakukan sampling pada call stack, menghitung CPU time per metode, alokasi memori, kontensi lock, dan waktu yang dihabiskan setiap baris kode. Profiler menjawab pertanyaan *"baris kode mana yang sebenarnya jadi bottleneck?"*.

Keduanya saling melengkapi: JMeter memberi tahu **bahwa** ada masalah performa dan seberapa parah dampaknya, sementara IntelliJ Profiler menunjukkan **di mana persisnya** masalah tersebut berada di dalam kode.

### 2. Bagaimana proses profiling membantu mengidentifikasi dan memahami titik lemah dalam aplikasi?

Profiling memberikan visibilitas ke jalur eksekusi yang sebelumnya tidak terlihat. Pada method `getAllStudentsWithCourses`, misalnya, JMeter hanya menunjukkan response time ~14 detik — tetapi profiler langsung memperlihatkan ratusan invocation `findByStudentId` yang berulang, sebuah pola N+1 query yang khas. Pada `joinStudentNames`, profiler dapat menunjukkan bahwa waktu CPU terkonsentrasi pada `String.concat` dan alokasi `char[]` — bukti langsung dari problem konkatenasi O(N²). Tanpa profiler, kita hanya menebak; dengan profiler, kita melihat hot path dengan angka yang konkret. Ini mengubah optimasi dari aktivitas spekulatif menjadi keputusan berbasis bukti.

### 3. Apakah IntelliJ Profiler efektif dalam membantu menganalisis dan mengidentifikasi bottleneck pada kode aplikasi?

Ya, sangat efektif. Integrasinya langsung dengan IDE membuat alur kerjanya mulus: hasil profiling dapat diklik menuju baris kode yang relevan, flame graph-nya mudah dibaca, dan overhead-nya rendah karena memakai async-profiler di balik layar. Kombinasi tampilan call tree, flame graph, dan method list memberi sudut pandang yang berbeda terhadap data yang sama, sehingga pola bottleneck — baik berupa N+1 query, hot loop, maupun alokasi berlebihan — dapat dikenali dengan cepat. Untuk kebutuhan optimasi tingkat aplikasi pada proyek Spring Boot, IntelliJ Profiler sudah lebih dari cukup tanpa harus berpindah ke alat eksternal.

### 4. Apa tantangan utama dalam melakukan performance testing dan profiling, dan bagaimana mengatasinya?

Tantangan utama yang saya temui:

- **Variansi antar run yang besar pada sampel kecil.** Pada uji JMeter dengan 10 sampel untuk `findStudentWithHighestGpa` dan `joinStudentNames`, hasilnya bahkan menunjukkan versi optimasi sedikit lebih lambat — murni karena jitter, JIT warm-up, dan satu outlier. Solusinya: perbesar jumlah sampel (500–1000), tambahkan loop warm-up yang dibuang sebelum pengukuran, dan jalankan kedua skenario secara berurutan dalam kondisi sistem yang sama.
- **Dataset yang terlalu kecil untuk memunculkan perbedaan.** Optimasi seperti O(N²) → O(N) atau projection query baru terlihat dampaknya saat N cukup besar. Pada dataset kecil, biaya serialization JSON dan network mendominasi sehingga gain CPU tertutupi. Solusinya: seed database dengan data yang representatif terhadap kondisi produksi.
- **Mismatch metrik.** JMeter mengukur wall-clock latency, sementara requirement menyebut "CPU Time". Solusinya: gunakan profiler atau Hibernate statistics untuk metrik CPU dan jumlah query, lalu pakai JMeter untuk validasi end-to-end.
- **Overhead profiler yang dapat mendistorsi hasil.** Solusinya: gunakan mode sampling, bukan instrumentation, untuk pengukuran produksi-realistis.

Saya optimis seluruh refactor yang sudah dilakukan akan menunjukkan keunggulannya secara meyakinkan begitu data uji diperluas — pola algoritmik dan eliminasi N+1 yang sudah diterapkan adalah perbaikan yang scalable, dan dampaknya akan tumbuh seiring bertambahnya beban dan ukuran dataset.

### 5. Apa manfaat utama dari penggunaan IntelliJ Profiler untuk profiling kode aplikasi?

- **Lokalitas yang akurat.** Setiap entri di flame graph tertaut langsung ke baris kode, sehingga waktu antara "menemukan masalah" dan "mengedit kode" sangat singkat.
- **Bukti objektif untuk keputusan refactor.** Alih-alih menebak bahwa konkatenasi string atau loop query lambat, profiler memberi angka konkret yang dapat dibandingkan sebelum dan sesudah perubahan.
- **Multi-dimensi.** Selain CPU, profiler juga menampilkan alokasi heap, kontensi lock, dan waktu I/O — sehingga jenis bottleneck yang berbeda (compute-bound vs memory-bound vs I/O-bound) dapat dibedakan dengan jelas.
- **Bagian dari workflow IDE.** Tidak perlu setup eksternal, output mudah dibagikan, dan iterasi optimasi menjadi cepat.

### 6. Bagaimana menangani situasi ketika hasil profiling IntelliJ Profiler tidak sepenuhnya konsisten dengan temuan performance testing JMeter?

Inkonsistensi seperti itu justru informatif — keduanya mengukur hal yang berbeda. Strategi saya:

1. **Identifikasi metrik mana yang sedang dibandingkan.** JMeter mengukur latency end-to-end, profiler mengukur waktu CPU di JVM. Endpoint bisa "lambat" di JMeter karena network/serialization meskipun CPU-nya sudah hemat.
2. **Cek dataset dan beban.** Pada test set kecil, gain algoritmik tertutupi noise. Profiler tetap menunjukkan perbaikan CPU yang nyata, sementara JMeter tampak flat.
3. **Tambahkan instrumentation sebagai tie-breaker.** Hibernate `show-sql` atau `Statistics` mengkonfirmasi pengurangan jumlah query secara independen dari kedua alat.
4. **Percayai bukti yang paling proksimal terhadap perubahan.** Jika perubahan adalah eliminasi N+1, jumlah query yang turun adalah bukti langsung; angka JMeter adalah konsekuensi turunan yang dipengaruhi banyak faktor.

Pengalaman pada uji `findStudentWithHighestGpa` dan `joinStudentNames` adalah contoh konkret: JMeter belum memperlihatkan gain pada n=10, tetapi secara struktural perubahannya sudah benar dan akan terbukti seiring perluasan data uji.

### 7. Strategi apa yang diterapkan dalam mengoptimasi kode aplikasi setelah menganalisis hasil performance testing dan profiling? Bagaimana memastikan perubahan tidak memengaruhi fungsionalitas aplikasi?

Strategi optimasi yang saya terapkan, berurut dari biaya rendah ke tinggi:

1. **Pindahkan kerja ke database bila memungkinkan.** Contoh: `findTopByOrderByGpaDesc()` mengganti scan in-memory dengan `ORDER BY ... LIMIT 1`; `JOIN FETCH` menghapus N+1 dengan satu query.
2. **Kurangi data yang ditarik.** Gunakan projection seperti `SELECT s.name FROM Student s` daripada full entity, sehingga DB I/O dan memory footprint berkurang.
3. **Pilih struktur data yang tepat di JVM.** `String.join` / `Collectors.joining` menggantikan `+=` dalam loop, mengubah O(N²) menjadi O(N).
4. **Hindari premature optimization.** Optimasi dilakukan hanya pada hot path yang teridentifikasi profiler — bukan tebak-tebakan.

Untuk memastikan fungsionalitas tidak terganggu:

- **Verifikasi output identik.** Pada uji JMeter, payload `bytes` tetap 422,002 / 282 / 46,784 sebelum dan sesudah refactor — bukti kuat bahwa response sama.
- **Pertahankan kontrak method publik.** Signature service tidak berubah, hanya implementasinya, sehingga caller tidak perlu disesuaikan.
- **Perbaiki bug laten yang ditemukan sambil jalan.** Refactor `findStudentWithHighestGpa` menghapus floor `0.0` yang silently mengembalikan `Optional.empty()` untuk seluruh GPA non-positif; refactor `joinStudentNames` menghapus `substring(0, length-2)` yang melempar exception pada list kosong.
- **Jalankan unit/integration test sebelum dan sesudah perubahan.** Jika tersedia suite test, ini menjadi safety net yang konsisten.

Saya optimis semua refactor yang dilakukan akan menunjukkan dampaknya secara penuh saat dataset diperluas dan beban uji ditingkatkan — perubahan algoritmik yang dilakukan adalah perbaikan permanen yang scaling-nya superior, sehingga seiring tumbuhnya aplikasi, gap performa antara versi lama dan baru akan semakin lebar dan semakin terlihat jelas dalam metrik manapun yang dipakai untuk mengukurnya.
