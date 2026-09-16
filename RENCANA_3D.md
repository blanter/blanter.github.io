# Rencana Migrasi Render 2D → 3D (Three.js)

Dokumen ini adalah acuan implementasi. Dikerjakan **bertahap per fase**; setiap fase
harus bisa dijalankan dan dimainkan sebelum lanjut ke fase berikutnya.

---

## 1. Kondisi Saat Ini

Seluruh game ada di `index.html` (~6.5k baris, satu file):

| Lapisan | Isi | Nasib di versi 3D |
|---|---|---|
| UI HTML + Tailwind | HUD, store, AI chat, menu, modal lore | **Dipertahankan apa adanya** |
| Canvas 2D `#gameCanvas` | Semua penggambaran dunia & entity | **Diganti Three.js** |
| Class entity | `Player`, `Terminator`, `Survivor`, `Colossus`, `Bullet`, `Grenade`, `Hovercar`, `Barrel`, `Pickup`, `Lamp`, `LoreObject`, `Particle`, ... | Data & `update()` tetap; `draw()` diganti |
| Logika dunia | `resolveCollision`, `hasLineOfSight`, worldgen `walls/floors/buildings`, AI state machine, wave system, ekonomi | **Tidak diubah sama sekali** |
| Efek layar | `drawRoofs`, overlay `isPowerOut`, `drawMinimap`, `drawCrosshair`, `drawAIEffects` | Sebagian jadi 3D, sebagian tetap canvas 2D overlay |

### Prinsip utama

> **Logika tetap 2D. Hanya presentasi yang jadi 3D.**

Semua posisi tetap `(x, y)` dalam koordinat dunia lama (map 3000–5000 px).
Collision tetap AABB/lingkaran di bidang datar. Tidak ada fisika 3D, tidak ada
gravitasi, tidak ada raycast untuk gameplay. Ini menjamin gameplay, balancing,
save file, dan semua angka damage tetap identik.

### Konvensi koordinat (WAJIB konsisten)

```
Dunia 2D          →  Three.js
x                 →  position.x
y                 →  position.z      (bukan .y!)
"ke atas layar"   →  -z
tinggi/ketinggian →  position.y      (sumbu baru, 0 = lantai)
angle (radian)    →  mesh.rotation.y = -angle
1 pixel dunia     →  1 unit Three.js  (skala 1:1, jangan diubah)
```

Helper wajib dipakai di mana-mana, jangan tulis manual:

```js
function toWorld3(x, y, h = 0) { return new THREE.Vector3(x, h, y); }
function syncMesh(mesh, ent, h = 0) {
  mesh.position.set(ent.x, h, ent.y);
  if (ent.angle !== undefined) mesh.rotation.y = -ent.angle;
}
```

---

## 2. Arsitektur Target

```
index.html
├── UI HTML (Tailwind)          ← tidak berubah
├── <canvas id="gameCanvas3D">  ← renderer Three.js (z-index 0)
├── <canvas id="overlayCanvas"> ← canvas 2D untuk minimap + crosshair (z-index 1)
└── <script>
    ├── R3 (namespace render 3D)   ← BARU
    │   ├── R3.init()              scene, camera, renderer, lighting
    │   ├── R3.buildStaticWorld()  lantai, tembok, gedung, atap
    │   ├── R3.registry            Map<entity, mesh>
    │   ├── R3.sync(dt)            samakan mesh dengan state entity
    │   └── R3.render()
    ├── Factory mesh               ← BARU (makePlayerMesh, makeTerminatorMesh, ...)
    └── Logika game lama           ← tetap
```

`R3.registry` adalah `Map` dari objek entity ke `THREE.Object3D`. Tiap frame:
entity yang baru muncul dibuatkan mesh, entity yang hilang dari array mesh-nya
di-dispose. Ini menghindari perlunya mengubah setiap `push`/`splice` yang
tersebar di seluruh file.

### Arah kamera & "style"

Game ini bergaya top-down cyberpunk neon. Supaya *style* tidak hilang:

- Kamera **perspective FOV 45°**, menengok ke bawah dengan kemiringan **~60°**
  dari horizontal (bukan 90° penuh) — memberi kedalaman tapi tetap terbaca
  sebagai top-down. Kamera mengikuti `player` persis seperti variabel `camera`
  lama.
- Material **`MeshStandardMaterial`** dengan `emissive` untuk semua garis neon,
  ditambah **UnrealBloomPass** (EffectComposer) supaya glow `shadowBlur` versi
  2D punya padanan 3D yang lebih bagus.
- Palet warna diambil **persis** dari kode lama: `#0b0f19` background,
  `#1e293b` tembok, `#38bdf8` neon cyan, `#ef4444` merah musuh,
  `#f472b6` visor magenta, `#a855f7` plasma.
- Fog `THREE.Fog('#0b0f19', ...)` menggantikan kesan gelap di tepi map.

---

## 3. Fase Implementasi

Setiap fase = satu commit yang bisa dimainkan.

### Fase 0 — Persiapan (tanpa perubahan visual)

- [ ] Backup: `cp index.html backup/pre-3d.html`
- [ ] Tambah import Three.js via ES module importmap (versi **pin**, mis. `three@0.180.0`)
      dari `cdn.jsdelivr.net`, termasuk `OrbitControls` (debug), `EffectComposer`,
      `RenderPass`, `UnrealBloomPass`.
- [ ] Tambah `<canvas id="gameCanvas3D">` di belakang canvas lama, dan CSS z-index.
- [ ] Buat namespace `R3` kosong dengan `init/sync/render` yang belum melakukan apa-apa.
- **Kriteria selesai:** game jalan 100% seperti sebelumnya, tidak ada regresi, console bersih.

### Fase 1 — Scene, kamera, dan lantai

- [ ] `R3.init()`: `WebGLRenderer({ antialias:true })`, `setPixelRatio(min(dpr,2))`,
      resize handler yang memanggil `camera.updateProjectionMatrix()`.
- [ ] Lantai: satu `PlaneGeometry(currentMapW, currentMapH)` rotasi `-PI/2`,
      diposisikan di tengah map, material gelap `#0b0f19`.
- [ ] Grid neon: `THREE.GridHelper` ukuran map, spacing 100 (sama dengan
      `drawEnvironment`), warna `rgba(14,165,233,0.05)` → opacity 0.05.
- [ ] Kamera follow: `camera.position.set(player.x, CAM_HEIGHT, player.y + CAM_BACK)`,
      `camera.lookAt(player.x, 0, player.y)`. Nilai awal: `CAM_HEIGHT = 620`,
      `CAM_BACK = 360` (setara area pandang canvas 2D lama, sesuaikan saat playtest).
- [ ] Lighting: `AmbientLight(#334155, 0.6)` + `DirectionalLight` dari atas-serong
      untuk shadow arah konsisten.
- [ ] Render 3D **di belakang**, canvas 2D lama masih menggambar semuanya di atasnya
      (masih terlihat 2D). Ini memudahkan perbandingan posisi.
- **Kriteria selesai:** lantai 3D bergerak sinkron persis dengan dunia 2D di atasnya.

### Fase 2 — Dunia statis (tembok, gedung, atap, lantai ruangan)

- [ ] `R3.buildStaticWorld()` dipanggil setelah worldgen dan setelah `mapExpanded`.
- [ ] `walls[]` → `BoxGeometry(w, WALL_H, h)` di `(x + w/2, WALL_H/2, y + h/2)`.
      `WALL_H = 120`. Pakai **satu material bersama** per jenis tembok, dan
      gabungkan dengan `InstancedMesh` (ratusan tembok → 1 draw call).
- [ ] Jenis khusus: `fence` (tinggi 160, tipis), `gate`/`door` (material emissive
      merah, `visible = w.closed`, dianimasikan naik/turun saat buka-tutup —
      menggantikan flicker `Math.random()` yang lama), tembok destructible
      (opacity ikut `hp/maxHp` seperti kode lama).
- [ ] `floors[]` → plane tipis pada `y = 0.5` supaya tidak z-fighting dengan lantai utama.
- [ ] `buildings[]` → atap `BoxGeometry` pada `y = WALL_H`.
- [ ] Port `drawRoofs()`: atap gedung tempat player berada → `material.opacity = 0.12`
      + `transparent = true`; gedung yang sudah dijelajahi 0.95. Efek "gelapkan
      luar gedung" diganti dengan menurunkan intensitas `AmbientLight` +
      `SpotLight` interior saat `playerInsideBuilding` — lebih natural di 3D.
- [ ] Hapus `cacheStaticMap()`/`cacheCanvas` dari jalur render (geometri statis
      sudah jadi cache alaminya).
- **Kriteria selesai:** navigasi dunia terasa benar; player tidak pernah tembus
  tembok yang terlihat, dan tembok yang terlihat sama posisinya dengan collision.

### Fase 3 — Player 3D

- [ ] `makePlayerMesh()` — `THREE.Group` rakitan primitif, meniru bentuk dari
      `Player.draw()` lama:
      - torso: `BoxGeometry` rounded (`RoundedBoxGeometry` atau box + bevel) `#0f172a`
      - trim neon cyan `#38bdf8` sebagai emissive edge
      - kepala: `SphereGeometry` `#fdbcb4`
      - visor: box tipis emissive `#f472b6` menghadap `+x` lokal
      - dua lengan kapsul, senjata box di tangan kanan
- [ ] Rotasi: `group.rotation.y = -player.angle` (angle tetap dihitung dari
      `worldMouse` seperti sebelumnya — perlu **raycast mouse ke plane y=0**
      untuk mendapat `worldMouse` yang benar di perspektif 3D; lihat §4.1).
- [ ] Animasi: bob jalan sederhana (`position.y = sin(t)*2` saat bergerak),
      recoil lengan saat menembak (port dari variabel `recoil` lama),
      muzzle flash = `PointLight` pendek + sprite additive.
- [ ] Senjata mengikuti `player.weapon` (warna & bentuk per senjata seperti kode lama,
      termasuk glow ungu untuk Plasma).
- **Kriteria selesai:** menembak ke arah kursor akurat di semua sudut layar.

### Fase 4 — Musuh 3D

Empat tipe di `Terminator` + `Colossus`. Tiap tipe punya siluet berbeda supaya
tetap bisa dibaca sekilas seperti versi 2D:

- [ ] `STALKER` — badan box gelap `#dc2626`, **4 kaki** kapsul dengan animasi
      langkah prosedural (fase sinus per kaki, offset 0/π).
- [ ] `INTERCEPTOR` — lebih kecil & ramping, **6 kaki**, gerak cepat, emissive `#f87171`.
- [ ] `DRONE` — **melayang** pada `y = 60`, tanpa kaki, rotor/ring berputar,
      mata laser emissive `#38bdf8`, bob vertikal.
- [ ] `BOSS_TANK` — besar (radius 45), badan berat `#450a0a`, lampu merah berdenyut.
- [ ] `Colossus` — skala terbesar, port efek visual khususnya dari class lama.
- [ ] Mata/scanner: `PointLight` merah kecil ber-intensitas mengikuti `state`
      (`WANDER` redup → `CHASE` terang), menggantikan indikator 2D.
- [ ] Health bar & indikator sinyal hive: `Sprite` yang selalu menghadap kamera
      (`SpriteMaterial` dengan tekstur di-generate dari canvas kecil), posisi
      di atas kepala. **Jangan** pakai HTML overlay (mahal untuk puluhan musuh).
- [ ] `Survivor` (sekutu) — bentuk mirip player tapi palet kuning `#facc15`;
      state `isBroken` → pose tumbang + percikan.
- **Kriteria selesai:** 60+ musuh di layar tetap ≥50 FPS. Jika tidak, lanjut ke §5.

### Fase 5 — Proyektil, partikel, dan efek

- [ ] `Bullet` — `InstancedMesh` kapsul emissive (pool 500 sudah ada, tinggal
      dipetakan 1:1 ke instance; `active=false` → `scale 0`). Peluru melayang
      pada `y = 25` (setinggi senjata).
- [ ] `Grenade` — mesh bola dengan lintasan **parabola visual**: `y` dihitung dari
      progres lempar (murni kosmetik, tidak memengaruhi logika ledakan).
- [ ] `Particle` — `THREE.Points` dengan `BufferGeometry` + atribut warna,
      satu draw call untuk 1000 partikel. Pool lama dipakai apa adanya.
- [ ] `Shockwave` — ring `TorusGeometry` mengembang, atau decal additive di lantai.
- [ ] `SparkDecal` — plane additive di `y = 1`, fade `life` seperti sebelumnya.
- [ ] `Fog` (patch kabut) — `Sprite` besar transparan / volumetrik sederhana.
- [ ] `Lamp` + `isPowerOut` — ini jadi **keunggulan besar 3D**: hapus seluruh
      trik `darkCanvas` + `destination-out`, ganti dengan mematikan/menyalakan
      `PointLight` tiap lampu dan menurunkan `AmbientLight`. Senter player =
      `SpotLight` yang mengikuti `player.angle`.
- [ ] `Barrel`, `Pickup`, `LoreObject`, `HiveSignal`, `spawnAIStructure` —
      mesh sederhana + animasi bob/putar yang sudah ada di kode lama.
- [ ] `Hovercar` — melayang `y = 20`, miring saat belok (`rotation.z` dari
      kecepatan belok), thruster emissive di belakang.
- **Kriteria selesai:** semua efek lama punya padanan; tidak ada `ctx.` tersisa
  untuk dunia.

### Fase 6 — UI, overlay, dan pembersihan

- [ ] `drawMinimap()` dan `drawCrosshair()` → pindah ke `#overlayCanvas` 2D.
      Kodenya **tidak perlu diubah**, hanya ganti `ctx` → `overlayCtx` dan
      pastikan di-clear tiap frame.
- [ ] `drawGeneratorArrow()`, `drawHiveMindUI()`, `drawAIEffects()` → overlay 2D
      (arah panah dihitung dari proyeksi kamera 3D, bukan `camera.x/y` lama).
- [ ] Hapus `#gameCanvas` lama, `cacheCanvas`, `cacheStaticMap()`, dan semua
      method `draw()` yang sudah mati.
- [ ] Ganti semua pemakaian variabel `camera` lama yang tersisa.
- [ ] Verifikasi save/load (`saveGame`) masih kompatibel — seharusnya ya, karena
      tidak ada field baru yang di-persist.
- **Kriteria selesai:** `grep -c "ctx\." index.html` hanya menyisakan overlay UI.

### Fase 7 — Polish & performa

- [ ] Bloom (`UnrealBloomPass`, strength ~0.8, threshold ~0.6) untuk semua neon.
- [ ] Shadow map: `PCFSoftShadowMap`, hanya `DirectionalLight` yang cast,
      `shadow.mapSize 2048`, dan **batasi** `castShadow` ke player/musuh saja.
- [ ] Frustum culling otomatis + jangan `sync()` entity yang jauh di luar layar.
- [ ] Opsi grafis di menu Pause: Low (tanpa bloom & shadow) / Medium / High —
      penting karena target awalnya game 2D yang ringan.
- [ ] Playtest balancing kamera: pastikan jarak pandang ke musuh **tidak lebih
      pendek** dari versi 2D, karena `detectionRadius = 550` diseimbangkan
      terhadap ukuran layar lama.

---

## 4. Titik Risiko yang Harus Diperhatikan

### 4.1 `worldMouse` — paling kritis

Versi lama: `worldMouse.x = mouse.x + camera.x`. Ini **tidak berlaku lagi** di
perspektif 3D. Harus diganti raycast:

```js
const ndc = new THREE.Vector2(
  (mouse.x / innerWidth) * 2 - 1,
  -(mouse.y / innerHeight) * 2 + 1
);
raycaster.setFromCamera(ndc, camera);
const hit = new THREE.Vector3();
raycaster.ray.intersectPlane(groundPlane, hit);  // groundPlane: y = 0
worldMouse.x = hit.x; worldMouse.y = hit.z;
```

Semua sistem membaca `worldMouse` (aim, lempar granat, crosshair, interaksi),
jadi ini harus benar sebelum Fase 3 dianggap selesai. Catatan: idealnya raycast
ke plane setinggi dada (`y = 25`), bukan `y = 0`, supaya bidikan terasa pas.

### 4.2 Kamera mengubah field of view gameplay

Kamera miring memperlihatkan area berbeda dari kamera ortho lama. Musuh
dengan `detectionRadius = 550` bisa terasa lebih adil/tidak adil. **Jangan ubah
angka AI** — sesuaikan `CAM_HEIGHT` sampai area terlihat setara, lalu playtest.

### 4.3 Tembok tinggi menghalangi pandangan

Di 2D tembok tidak pernah menutupi apa pun. Di 3D, `WALL_H = 120` dengan kamera
miring akan menutupi musuh di baliknya. Mitigasi: tembok luar dibuat lebih pendek,
dan tembok yang berada **antara kamera dan player** dibuat semi-transparan
(deteksi via raycast kamera→player, sama pola dengan logika atap yang sudah ada).

### 4.4 Ukuran file

`index.html` sudah 314 KB. Setelah fase 6, **pecah** jadi:
`index.html`, `js/render3d.js`, `js/meshes.js`, `js/game.js` (ES modules).
Jangan lakukan ini sebelum fase 6 selesai — refactor dan migrasi render
sekaligus akan menyulitkan pelacakan bug.

### 4.5 Duplikasi kode yang sudah ada

Ada baris terduplikasi di `class Player` (mis. `this.ownedWeapons` dan
`this.inVehicle` di-set dua kali, sekitar baris 1735–1740). Bersihkan saat
menyentuh class tersebut di Fase 3.

---

## 5. Anggaran Performa

Target: **60 FPS** dengan 60 musuh + 200 peluru + 500 partikel.

| Kategori | Strategi | Draw call target |
|---|---|---|
| Tembok statis | `InstancedMesh` per material | ≤ 5 |
| Lantai & atap | merge geometry | ≤ 3 |
| Musuh | `InstancedMesh` per tipe **jika** >30 musuh, kalau tidak Group biasa | ≤ 8 |
| Peluru | `InstancedMesh` tunggal | 1 |
| Partikel | `THREE.Points` tunggal | 1 |
| Health bar | `Sprite` + shared material | ≤ 2 |
| **Total** | | **< 40** |

Aturan: **jangan pernah** membuat geometry/material di dalam loop render.
Semua di-cache di `R3.assets` dan di-clone/di-instance.

---

## 6. Checklist Regresi (jalankan tiap akhir fase)

- [ ] Gerak WASD + tabrakan tembok terasa sama
- [ ] Menembak akurat ke kursor di pojok layar
- [ ] Granat mendarat di titik yang dibidik
- [ ] Musuh mengejar/kehilangan jejak di jarak yang sama
- [ ] Masuk/keluar gedung: atap fade, interior terlihat
- [ ] Power outage: gelap + lampu senter berfungsi
- [ ] Naik/turun hovercar
- [ ] Store buka, beli senjata, ganti senjata
- [ ] Wave berikutnya spawn, boss muncul
- [ ] Save → reload → load: posisi & inventori utuh
- [ ] Minimap & crosshair tampil benar
- [ ] AI chat + semua efeknya (hack, repel, convert, teleport, medkit)
- [ ] Game over → kembali ke menu → main lagi tanpa kebocoran memori
      (cek `R3.registry.size` kembali ke 0)

---

---

## STATUS IMPLEMENTASI

Fase 0-6 **sudah diimplementasikan** di `index.html`. Backup versi 2D ada di
`backup/pre-3d.html`. Semua perubahan bersifat aditif: logika gameplay tidak
disentuh sama sekali (diverifikasi, lihat "Bukti" di bawah).

| Fase | Status | Catatan |
|---|---|---|
| 0 Persiapan | Selesai | importmap + modul loader, `#gameCanvas3D`, backup |
| 1 Scene & kamera | Selesai | lantai, grid neon, kamera follow, fog, lighting |
| 2 Dunia statis | Selesai | tembok ter-instance + garis neon, atap, pintu, lampu |
| 3 Player 3D | Selesai | mantel, visor, lengan, recoil, muzzle flash |
| 4 Musuh 3D | Selesai | 4 tipe Terminator, Colossus, Survivor, health bar sprite |
| 5 Proyektil & efek | Selesai | peluru instanced, partikel Points, shockwave, decal, kabut |
| 6 UI & pembersihan | Selesai | overlay 2D dipakai ulang, cache bitmap dimatikan |
| 7 Polish | Sebagian | bloom & shadow aktif; menu kualitas **belum** terpasang di UI |

### Cara kerja akhirnya

Canvas 2D lama **tidak dihapus** — ia dipakai ulang sebagai overlay transparan
untuk minimap, crosshair, dan teks hint, sehingga kode UI itu tidak perlu
diubah. Canvas 3D berada di belakangnya. Setiap `draw()` entity diawali
`if (R3.enabled) return;`, jadi mematikan `R3.enabled` mengembalikan game ke
render 2D sepenuhnya — jalur itu sudah diuji dan bekerja.

Dunia statis dibangun ulang otomatis saat `R3.staticSignature()` berubah
(map expand, secret base, restart), jadi tidak ada satu pun `push`/`splice`
di kode lama yang perlu disentuh. Entity dinamis disinkronkan lewat
`R3.registry` dengan penanda generasi.

### Tiga keputusan tambahan — semuanya terpasang

1. **`worldMouse` via raycast** (`R3.updateWorldMouse`). Raycast ke bidang
   setinggi dada (`AIM_HEIGHT = 25`), bukan lantai. `R3.updateCamera()`
   dipanggil **sebelum** raycast — kalau tidak, bidikan memakai posisi kamera
   frame sebelumnya dan meleset saat framerate turun.
2. **Skala pandang kamera dipertahankan.** `dist = (innerHeight/2) / tan(fov/2)`
   membuat area pandang di posisi player identik dengan versi 2D, sehingga
   `detectionRadius = 550` tetap seimbang. Tidak ada satu pun angka AI diubah.
3. **Tembok penghalang jadi transparan.** `R3.updateWallOcclusion()` memakai
   geometri (jangkauan halangan = `tinggi / tan(kemiringan)`) untuk memilih
   tembok di koridor antara player dan kamera, menyembunyikan instance aslinya
   dan menampilkan mesh "ghost" semi-transparan dari pool 16 buah.

### Dua bug yang ditemukan saat verifikasi (sudah diperbaiki)

- **Semua PointLight/SpotLight tidak terlihat.** Three.js r155+ memakai satuan
  fisik: iluminasi = `intensity / jarak²`. Dunia ini berskala piksel (jarak
  ratusan unit), jadi `intensity: 1.4` menghasilkan cahaya nyaris nol — lampu
  jalan dan senter saat mati listrik praktis mati. Diperbaiki dengan helper
  `LUX(terang, jarak) = terang * jarak²` pada semua lampu titik/sorot.
- **Transisi visual terikat `dt` gameplay**, yang di-nol-kan saat game dijeda,
  sehingga fade atap dan lampu membeku di menu/store. Sekarang memakai
  `R3._rdt` (delta waktu render sungguhan).

### Bukti verifikasi

Dijalankan di Chrome headless (WebGL via SwiftShader) terhadap build final:

- **Logika gameplay identik byte-per-byte** dengan `backup/pre-3d.html` untuk
  `resolveCollision`, `hasLineOfSight`, `Player.update`, konstruktor
  `Terminator`/`Colossus`, `Bullet`, `Grenade`, `Particle`, `Pickup.collect`,
  `Barrel.explode`, `saveGame`, dan biaya AI — 0 blok berubah.
- **Render**: 17 draw call, 43.810 triangle dengan 20 musuh + dunia statis penuh
  (target dokumen ini: <40 draw call).
- **Runtime**: 0 error/console-error selama sesi bermain otomatis.
- **Fallback 2D** (CDN Three.js sengaja digagalkan): game tetap jalan, 0 error,
  kanvas 2D terbukti tergambar.

Catatan jujur: **FPS belum diukur pada GPU sungguhan.** SwiftShader berjalan
~0,3 fps sehingga angka framerate dari pengujian ini tidak bermakna. Jumlah
draw call dan triangle di atas valid, tapi target 60 FPS di Fase 7 masih perlu
dicek di perangkat nyata.

### Putaran perbaikan kedua (setelah playtest)

Lima keluhan ditangani: frame patah-patah, kotak nyangkut di layar, efek
tembakan tidak terlihat, peta terlalu gelap, dan UI tidak konsisten.

**Bug yang diperbaiki**

- **Kotak nyangkut di layar.** `drawRoofs()` tidak ikut di-guard, jadi atap
  gedung tetap digambar di koordinat dunia ke overlay 2D yang sudah tidak
  punya transform kamera. Hasilnya atap `house1` (500,500 - 1000,900) muncul
  sebagai kotak diam di layar. Sekarang di-guard; fade atap sepenuhnya
  ditangani mesh 3D.
- **Tracer peluru tidak terlihat.** Geometry peluru tidak punya atribut
  `color`, padahal materialnya `vertexColors: true`. Atribut yang hilang
  terbaca nol di shader, jadi semua peluru hitam. Atribut putih ditambahkan
  agar `instanceColor` yang menentukan warnanya.
- **Pita cyan raksasa melintang layar.** Sinyal Hive memakai `TorusGeometry`
  yang diskalakan sampai radius 1500 — tabungnya ikut terskala jadi ~90 unit.
  Diganti `LineLoop`, yang tebalnya tetap berapa pun radiusnya.

**Performa**

- Bloom dirender di setengah resolusi (pass blur berlapis adalah biaya
  terbesar per frame).
- Pixel ratio dibatasi 1.5, bukan 2.
- Shadow map 2048 -> 1024, frustum 900 -> 700.
- Material musuh di-cache per tipe (`R3.enemyMats`), bukan dialokasikan per
  entity. Catatan jujur: Three.js men-dedupe *program shader* berdasarkan
  parameter, jadi ini **tidak** menghemat ratusan kompilasi seperti dugaan
  awal — yang dihemat adalah alokasi objek dan perpindahan render-state.
  Terukur: 60 musuh tetap hanya 31 program shader.
- LOD: musuh di luar radius 1250 tidak dianimasikan (`LOD_FAR_SQ`).
- Alokasi `new THREE.Object3D()` per frame di `animateLegs` dan
  `updateWallOcclusion` dihapus, diganti objek scratch bersama.

**Tampilan**

- Peta diterangkan: ambient 1.1 -> 1.6, sun 1.0 -> 1.5, hemi 0.7 -> 1.1,
  warna tanah dan tembok dinaikkan, kabut dimundurkan ke 1500-3400.
- Efek tembakan: muzzle flash berupa kerucut + cincin kejut yang memanjang
  lalu menciut, cahaya 3x lebih kuat, plus lapisan glow additif di belakang
  tracer.

**UI**

Seluruh HUD disatukan ke satu sistem panel (`.hud-panel`, `.hud-slot`,
`.hud-label`, `.hud-value`, `.hud-bar`) di `<style>`: bentuk sudut terpotong,
garis aksen tepi atas, dan tipografi label yang sama untuk semua elemen.
Yang berbeda hanya variabel CSS `--ac`. Radar digambar ulang di canvas dengan
bentuk dan warna yang sama persis dengan panel HTML lainnya. Warna aksen panel
HP kini mengikuti kondisi (cyan -> kuning -> merah).

**Menu kualitas grafis** sudah terpasang di layar Pause (Rendah/Sedang/Tinggi,
tersimpan di localStorage). Ini jawaban langsung untuk frame yang patah-patah:
"Rendah" mematikan bloom dan bayangan.

Verifikasi putaran ini: 0 error runtime dengan 40-60 musuh; ketiga level
kualitas berpindah dengan benar; `R3.reset()` mengembalikan registry ke 0;
mati listrik terbukti secara visual (dunia gelap, senter menyorot, lampu
jalan padam). FPS **masih belum** terukur di GPU sungguhan — alasannya sama
seperti sebelumnya.

### Yang belum dikerjakan

- Pemecahan `index.html` menjadi beberapa file (Fase 6, §4.4). File sekarang
  386 KB. Sengaja ditunda supaya migrasi render tidak bercampur dengan refactor.
- Checklist regresi manual di §6 — perlu dijalankan dengan tangan di browser.


## 7. Urutan Kerja yang Disarankan

```
Fase 0 → 1 → 2   : fondasi, paling banyak menentukan rasa akhirnya
Fase 3 → 4       : karakter & musuh (inti permintaan)
Fase 5           : efek, paling banyak baris tapi paling repetitif
Fase 6 → 7       : bersih-bersih & polish
```

Jangan menggabung fase. Tiap fase commit terpisah agar mudah `git revert`
kalau satu pendekatan ternyata merusak *feel* permainannya.
