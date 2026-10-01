# Evaluasi Keterbatasan YOLO pada Deteksi Objek Kecil Berkelompok

**Pengaruh Resolusi Input 640×640 vs 1280×1280 pada VisDrone2019-DET**

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1OSrupChmztcs8s-jjMuyXK7diRoSMSBo?usp=sharing)

Tugas Computer Vision (Assignment 02), **Opsi C — YOLO**
Penulis: **Royan Aditya** 

---

## Ringkasan

Repositori ini berisi notebook eksperimen yang menguji klaim pada paper *You Only Look Once* (Redmon et al., CVPR 2016), Bagian 2.4 (*Limitations of YOLO*): YOLO kesulitan mendeteksi **objek kecil yang berkelompok** karena representasi berbasis grid dan terbatasnya jumlah prediksi per sel.

Eksperimen ini **bukan** re-implementasi YOLOv1. Klaim tersebut diuji pada detektor satu tahap modern (**YOLO11n**) menggunakan citra udara dari drone (**VisDrone2019-DET**), lalu dibandingkan dua resolusi input.

### Pertanyaan riset

1. **RQ1.** Apakah kesulitan deteksi objek kecil yang dilaporkan pada YOLOv1 masih terlihat pada YOLO11n, berdasarkan AP untuk objek *small*, *medium*, dan *large*?
2. **RQ2.** Apakah menaikkan resolusi input dari 640 ke 1280 memperbaiki AP objek kecil, dan berapa biaya komputasinya?

### Kondisi eksperimen

| Kondisi | Model | Resolusi latih dan uji | Peran |
|---|---|---|---|
| A | YOLO11n (pretrained COCO) | 640×640 | Baseline |
| B | YOLO11n (pretrained COCO) | 1280×1280 | Intervensi |

Semua hiperparameter identik (30 epoch, tanpa early stopping, seed 42). Hanya resolusi input yang berbeda.

---

## Dataset

**VisDrone2019-DET** (Tianjin University), 10 kelas: `pedestrian`, `people`, `bicycle`, `car`, `van`, `truck`, `tricycle`, `awning-tricycle`, `bus`, `motor`.

| Split | Jumlah citra | Keterangan |
|---|---|---|
| Latih | 2.000 | Sub-sampel acak (seed 42) dari 6.471 citra, karena batas waktu Colab |
| Validasi | 548 | `val` |
| Uji | 1.610 | `test-dev`, tidak dipakai saat pelatihan |

Data diunduh otomatis dari GitHub Releases Ultralytics (sekitar 2,3 GB), lalu anotasi VisDrone dikonversi ke format YOLO. Baris anotasi dengan skor 0 (*ignored regions*) dan kategori *others* dibuang.

Ukuran objek mengikuti definisi COCO (piksel citra asli): *small* < 32², *medium* 32²–96², *large* > 96².

| Ukuran | Jumlah GT (test) | Proporsi |
|---|---|---|
| Small | 50.837 | ± 67,7% |
| Medium | 21.856 | ± 29,1% |
| Large | 2.409 | ± 3,2% |
| **Total** | **75.102** | |

---

## Metodologi

- **Pelatihan:** fungsi yang sama untuk kedua kondisi, hanya `imgsz` yang berbeda.
- **Evaluasi standar:** `model.val()` Ultralytics pada test-dev (`conf=0.001`, `iou=0.7`, `max_det=500`).
- **AP per ukuran objek:** dihitung dengan `pycocotools`, karena Ultralytics tidak menyediakannya.
- **Validasi silang:** mAP@0,5 dihitung ulang dengan fungsi AP manual (interpolasi semua titik ala VOC) dan dibandingkan dengan dua sumber lain.
- **Analisis kasus gagal:** recall sadar-kelas vs agnostik-kelas pada `conf ≥ 0.25`, `IoU ≥ 0.5`, serta contoh visual objek yang terlewat (FN).
- **Analisis grid YOLOv1:** menghitung persentase objek yang "terbuang" bila satu sel grid hanya boleh bertanggung jawab atas satu objek (S = 7, 20, 40, 80, 160). Fungsi `encode_yolov1_target` dan `yolov1_loss` diadaptasi dari notebook tutorial Sesi 3.

---

## Hasil

> Semua angka berasal dari output notebook (satu seed, satu run per kondisi, 30 epoch, GPU Tesla T4).

### Metrik standar pada test-dev (Ultralytics)

| Metrik | A: 640 | B: 1280 |
|---|---|---|
| mAP@0,5 | 0,2001 | 0,3249 |
| mAP@0,5:0,95 | 0,1079 | 0,1869 |
| Precision | 0,3016 | 0,4259 |
| Recall | 0,2510 | 0,3580 |

### AP per ukuran objek (pycocotools)

| Metrik | Ukuran | A: 640 | B: 1280 | Δ relatif |
|---|---|---|---|---|
| mAP@0,5 | Small | 0,0802 | 0,1808 | +125,5% |
| mAP@0,5 | Medium | 0,2847 | 0,4457 | +56,5% |
| mAP@0,5 | Large | 0,4546 | 0,5517 | +21,4% |
| mAP@0,5:0,95 | Small | 0,0328 | 0,0837 | +154,8% |
| mAP@0,5:0,95 | Medium | 0,1572 | 0,2698 | +71,6% |
| mAP@0,5:0,95 | Large | 0,2921 | 0,3965 | +35,7% |

Pada kedua resolusi, AP objek **small** merupakan yang terendah, dan kenaikan relatif terbesar dari resolusi 1280 juga terjadi pada objek small.

### Biaya komputasi

| | A: 640 | B: 1280 |
|---|---|---|
| Waktu latih (menit) | 27,32 | 60,88 |
| Total ms per citra | 18,92 | 24,88 |
| FPS | 52,8 | 40,2 |
| Parameter | 2,59 M | 2,59 M |

### Recall dan objek terlewat (`conf ≥ 0.25`, `IoU ≥ 0.5`)

| Ukuran | Recall A (sadar-kelas) | Recall B (sadar-kelas) | Recall A (agnostik) | Recall B (agnostik) |
|---|---|---|---|---|
| Small | 0,1787 | 0,3377 | 0,2156 | 0,3923 |
| Medium | 0,5928 | 0,7154 | 0,7004 | 0,8112 |
| Large | 0,7443 | 0,8082 | 0,8294 | 0,8809 |
| All | 0,3174 | 0,4627 | 0,3764 | 0,5299 |

Objek *small* menyumbang FN terbanyak (41.752 pada A dan 33.671 pada B), sehingga contoh kasus gagal difokuskan pada ukuran ini.

### Tabrakan grid (persentase GT terbuang pada satu objek per sel, test)

| S | 7 | 20 | 40 | 80 | 160 |
|---|---|---|---|---|---|
| GT terbuang (%) | 66,49 | 35,35 | 18,75 | 8,11 | 2,44 |

Pada grid kasar seperti YOLOv1 (S = 7), sekitar dua pertiga objek di data uji akan terbuang karena berbagi sel dengan objek lain. Rata-rata kepadatan citra uji adalah 46,6 objek per citra (maksimum 461).

### Perbandingan sumber mAP@0,5 (test)

| | Ultralytics | pycocotools | AP manual (VOC) |
|---|---|---|---|
| A: 640 | 0,2001 | 0,1744 | 0,1731 |
| B: 1280 | 0,3249 | 0,2974 | 0,2984 |

Angka dari tiap sumber berbeda karena prosedur interpolasi, urutan pencocokan, dan cara merata-ratakan kelas tidak sama. Perbandingan A vs B selalu dilakukan dalam pipeline yang sama.

---

## Cara menjalankan

1. Buka notebook melalui badge **Open in Colab** di atas, atau unggah `CVL_Assignment02.ipynb` ke Google Colab.
2. Pilih **Runtime → Change runtime type → GPU** (T4 atau lebih baik).
3. Jalankan **Runtime → Run all**. Dataset diunduh dan dikonversi otomatis.

Dependensi dipasang langsung di sel pertama:

```bash
pip install ultralytics pycocotools
```

Lingkungan yang tercatat pada output notebook: Ultralytics 8.4.166, PyTorch 2.11.0+cu128, GPU Tesla T4.

Parameter utama (satu tempat konfigurasi di sel setup):

| Parameter | Nilai |
|---|---|
| `MODEL_WEIGHTS` | `yolo11n.pt` |
| `EPOCHS` | 30 |
| `PATIENCE` | 1000 (early stopping dinonaktifkan secara praktis) |
| `BATCH_SIZE` | 2 |
| `TRAIN_SUBSET` | 2000 (`None` untuk memakai seluruh 6.471 citra) |
| `MAX_DET` | 500 |
| `SEED` | 42 |

> Catatan: pelatihan kondisi B (1280) memakan waktu lebih dari satu jam pada T4, dan prediksi dijalankan di subprocess terpisah agar RAM kernel tidak habis. Prediksi disimpan per citra sehingga proses dapat dilanjutkan bila terhenti.

---

## Struktur notebook

| Bagian | Isi |
|---|---|
| 1 | Setup dan reproduktibilitas (seed, konfigurasi, info lingkungan) |
| 2 | Unduh dataset, konversi anotasi, distribusi kelas dan ukuran objek |
| 3 | Ringkasan metode, formulasi YOLOv1, analisis tabrakan grid |
| 4 | Pelatihan dan evaluasi standar (Ultralytics) |
| 5 | Evaluasi AP per ukuran objek (pycocotools dan AP manual) |
| 6 | Analisis kasus gagal (recall sadar-kelas vs agnostik, contoh FN) |
| 7 | Ringkasan, keterbatasan, dan penyimpanan hasil |

## Luaran

Seluruh tabel dan gambar disimpan ke `/content/results` selama sesi Colab, antara lain:

- `summary_table.csv`, `standard_metrics_test.csv`, `size_specific_comparison.csv`
- `recall_by_size_at_conf.csv`, `fn_by_size.csv`, `tp_fp_fn_total.csv`
- `class_distribution.csv`, `size_distribution_by_split.csv`, `size_distribution_by_class.csv`, `grid_collision.csv`
- `dataset_split_summary.csv`, `environment.json`, `train_info_*.json`
- `figures/` (distribusi ukuran, contoh anotasi, perbandingan AP, kasus gagal)

---

## Keterbatasan

- Pelatihan hanya memakai sub-sampel 2.000 dari 6.471 citra, sehingga angka mutlak lebih rendah dibanding model yang dilatih penuh.
- Satu seed dan satu run per kondisi. GPU tidak sepenuhnya deterministik, jadi selisih kecil dapat masuk rentang noise.
- Tanpa early stopping dan epoch tetap (30): adil antar kondisi, tetapi kondisi 1280 mungkin belum konvergen (epoch terbaik pada kedua kondisi adalah epoch ke-30).
- Ambang ukuran COCO memakai piksel citra asli, sedangkan ukuran citra VisDrone bervariasi.
- `MAX_DET = 500` dapat memotong recall pada citra yang sangat padat.
- Kelas *ignored regions* dan *others* dibuang, dan evaluasi memakai test-dev (bukan test-challenge).
- Angka mAP dari Ultralytics, pycocotools, dan AP manual berbeda, sehingga sumber tiap angka harus dicantumkan.
- Hanya model YOLO11n yang diuji. Angka paper asli (mAP pada PASCAL VOC) tidak dapat dibandingkan langsung, yang dibandingkan hanya tren kualitatif.

## Pengembangan lanjutan

Tiling/SAHI saat inferensi, beberapa seed, bootstrap atas citra untuk selang kepercayaan AP, pelatihan pada seluruh 6.471 citra, serta pembanding model YOLO11s/11m.

## Referensi

- J. Redmon, S. Divvala, R. Girshick, A. Farhadi, "You Only Look Once: Unified, Real-Time Object Detection," CVPR 2016.
- D. Du et al., "VisDrone-DET2019: The Vision Meets Drone Object Detection in Image Challenge Results," Tianjin University.
- Ultralytics YOLO11 — https://docs.ultralytics.com
