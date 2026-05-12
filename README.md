# 📝 Latihan: Struktur Data Aplikasi Note-Taking

> **Struktur Data Lanjut — Advanced Linked Lists (Chapter 9)**

---

## 📋 Soal

> *"Rancang struktur data untuk aplikasi note-taking yang mendukung multiple tags per note (multi-linked by tag), chronological dan alphabetical views (doubly linked sorted), dan sync status tracking (circular buffer for recent changes)."*

---

## 🗺️ Peta Solusi

| Kebutuhan | Struktur | Alasan |
|---|---|---|
| Multiple tags per note | Multi-Linked List | Satu node masuk banyak chain sekaligus |
| Chronological & alphabetical view | Doubly Linked List (sorted) | Traversal dua arah, dua urutan berbeda |
| Sync status tracking | Circular Buffer | Round-robin, O(1) push, memori tetap |

---

## 🔧 Struktur Node

```python
class NoteData:
    def __init__(self, title, content, tags, timestamp):
        self.title     = title
        self.content   = content
        self.tags      = tags       # list[str]
        self.timestamp = timestamp  # datetime

class NoteNode:
    def __init__(self, data):
        self.data = data

        self.nextChron = None   # chain waktu →
        self.prevChron = None   # chain waktu ←
        self.nextAlpha = None   # chain A-Z →
        self.prevAlpha = None   # chain A-Z ←

        self.tagNext = {}       # { "study": NoteNode }
        self.tagPrev = {}       # { "study": NoteNode }
```

---

## 🏷️ Fitur 1 — Multi-Linked by Tag

Setiap tag punya chain sendiri. Satu node bisa masuk banyak chain tanpa duplikasi data.

```
Note A [study, python]      Chain #study  : A ↔ B
Note B [study, math]    →   Chain #python : A ↔ C
Note C [python, tips]       Chain #math   : B
                            Chain #tips   : C
```

```python
def _insert_tag(self, node, tag):
    node.tagNext[tag] = None
    node.tagPrev[tag] = None
    if tag not in self.tagHeads:
        self.tagHeads[tag] = self.tagTails[tag] = node
    else:
        tail = self.tagTails[tag]
        tail.tagNext[tag] = node
        node.tagPrev[tag] = tail
        self.tagTails[tag] = node

def view_by_tag(self, tag):
    result, cur = [], self.tagHeads.get(tag)
    while cur:
        result.append(cur.data.title)
        cur = cur.tagNext.get(tag)
    return result

def _remove_from_tag(self, node, tag):
    p, n = node.tagPrev.get(tag), node.tagNext.get(tag)
    if p: p.tagNext[tag] = n
    else: self.tagHeads[tag] = n
    if n: n.tagPrev[tag] = p
    else: self.tagTails[tag] = p
```

---

## 🔄 Fitur 2 — Doubly Linked Sorted

Dua chain sorted berbeda pada kumpulan node yang sama. Traversal balik O(n) via pointer `prev`.

```
Chain Chron : [Jan 1] ↔ [Jan 5] ↔ [Jan 10] ↔ [Jan 15]
Chain Alpha : [Binary] ↔ [Intro] ↔ [Linked] ↔ [OOP]
              ↑ node yang sama, dua urutan berbeda ↑
```

```python
def _insert_chron(self, node):
    if not self.headChron:
        self.headChron = self.tailChron = node; return
    if node.data.timestamp <= self.headChron.data.timestamp:
        node.nextChron = self.headChron
        self.headChron.prevChron = node
        self.headChron = node; return
    if node.data.timestamp >= self.tailChron.data.timestamp:
        node.prevChron = self.tailChron
        self.tailChron.nextChron = node
        self.tailChron = node; return
    cur = self.headChron
    while cur and cur.data.timestamp < node.data.timestamp:
        cur = cur.nextChron
    node.nextChron = cur
    node.prevChron = cur.prevChron
    cur.prevChron.nextChron = node
    cur.prevChron = node

# _insert_alpha identik, ganti timestamp→title dan Chron→Alpha

def view_chronological(self):
    result, cur = [], self.headChron
    while cur:
        result.append((cur.data.timestamp, cur.data.title))
        cur = cur.nextChron
    return result

def view_chronological_reverse(self):
    result, cur = [], self.tailChron    # mulai dari tail
    while cur:
        result.append((cur.data.timestamp, cur.data.title))
        cur = cur.prevChron
    return result

def _remove_chron(self, node):
    if node.prevChron: node.prevChron.nextChron = node.nextChron
    else:              self.headChron = node.nextChron
    if node.nextChron: node.nextChron.prevChron = node.prevChron
    else:              self.tailChron = node.prevChron
```

---

## 🔁 Fitur 3 — Circular Buffer (Sync Tracking)

Array melingkar ukuran tetap. Saat penuh, entri terlama tertimpa otomatis. Push selalu O(1).

```
     [0]      [1]      [2]      [3]      [4]      [5]
   del A    add B    edit C   add D   (kosong) (kosong)
   synced   synced   pending  pending
                       ↑                  ↑
                      head               tail
```

```python
BUFFER_SIZE = 10

class ChangeRecord:
    def __init__(self, action, note_title, timestamp):
        self.action     = action        # "ADD" | "EDIT" | "DELETE"
        self.note_title = note_title
        self.timestamp  = timestamp
        self.synced     = False

class SyncBuffer:
    def __init__(self):
        self.buf   = [None] * BUFFER_SIZE
        self.head  = 0
        self.tail  = 0
        self.count = 0

    def push(self, change):                             # O(1)
        self.buf[self.tail] = change
        self.tail = (self.tail + 1) % BUFFER_SIZE
        if self.count == BUFFER_SIZE:
            self.head = (self.head + 1) % BUFFER_SIZE  # timpa terlama
        else:
            self.count += 1

    def get_pending(self):                              # O(c)
        return [
            self.buf[(self.head + i) % BUFFER_SIZE]
            for i in range(self.count)
            if not self.buf[(self.head + i) % BUFFER_SIZE].synced
        ]

    def mark_synced(self, note_title):
        for i in range(self.count):
            e = self.buf[(self.head + i) % BUFFER_SIZE]
            if e and e.note_title == note_title:
                e.synced = True
```

---

## 🔩 Integrasi — NoteApp

```python
class NoteApp:
    def __init__(self):
        self.headChron = self.tailChron = None
        self.headAlpha = self.tailAlpha = None
        self.tagHeads  = {}
        self.tagTails  = {}
        self.syncBuf   = SyncBuffer()

    def add_note(self, title, content, tags, timestamp):
        node = NoteNode(NoteData(title, content, tags, timestamp))
        self._insert_chron(node)
        self._insert_alpha(node)
        for tag in tags:
            self._insert_tag(node, tag)
        self.syncBuf.push(ChangeRecord("ADD", title, timestamp))

    def delete_note(self, node):
        self._remove_chron(node)
        self._remove_alpha(node)
        for tag in node.data.tags:
            self._remove_from_tag(node, tag)
        self.syncBuf.push(ChangeRecord("DELETE", node.data.title, node.data.timestamp))
```

---

## 📊 Analisis Kompleksitas

| Operasi | Waktu | Keterangan |
|---|---|---|
| `add_note()` | O(n + t) | n = total note, t = jumlah tag |
| `delete_note()` | O(t) | O(1) per chain chron/alpha |
| `view_by_time()` | O(n) | Traversal maju atau mundur |
| `view_by_alpha()` | O(n) | Traversal maju atau mundur |
| `view_by_tag(tag)` | O(k) | k = note bertag tersebut saja |
| `sync_push()` | **O(1)** | Selalu konstan |
| `get_pending()` | O(c) | c ≤ BUFFER_SIZE |

**Ruang:** O(n × t) — pointer per node sebanyak jumlah tagnya.

---

## 💻 Contoh Output

```
$ python note_app.py

[ADD] "Intro Python"      | tags: [study, python]    | Jan 01 08:00
[ADD] "Belajar OOP"       | tags: [study, python]    | Jan 03 09:30
[ADD] "Algoritma Sorting" | tags: [study, algorithm] | Jan 05 14:00
[ADD] "List Comprehension"| tags: [python, tips]     | Jan 07 10:15
[ADD] "Binary Search"     | tags: [algorithm, tips]  | Jan 10 16:45
[ADD] "Decorator Python"  | tags: [python, tips]     | Jan 12 11:00

—— view_chronological() ————————————————————
#1  Jan 01 08:00  |  Intro Python
#2  Jan 03 09:30  |  Belajar OOP
#3  Jan 05 14:00  |  Algoritma Sorting
#4  Jan 07 10:15  |  List Comprehension
#5  Jan 10 16:45  |  Binary Search
#6  Jan 12 11:00  |  Decorator Python

—— view_alphabetical() ————————————————————
#1  Algoritma Sorting
#2  Belajar OOP
#3  Binary Search
#4  Decorator Python
#5  Intro Python
#6  List Comprehension

—— view_by_tag("python") ——————————————————
→ Intro Python
→ Belajar OOP
→ List Comprehension
→ Decorator Python
4 notes ditemukan.

—— view_by_tag("algorithm") ———————————————
→ Algoritma Sorting
→ Binary Search
2 notes ditemukan.

—— syncBuf.status() ————————————————————————
[0]  ADD   Intro Python      Jan 01  ✅ synced
[1]  ADD   Belajar OOP       Jan 03  ✅ synced
[2]  ADD   Algoritma Sorting Jan 05  ⏳ pending
[3]  ADD   List Comprehension Jan 07  ⏳ pending
[4]  ADD   Binary Search     Jan 10  ⏳ pending
[5]  ADD   Decorator Python  Jan 12  ⏳ pending
head=0  tail=6  count=6/10

—— setelah delete("List Comprehension") ————
Dihapus dari chain: #python, #tips, chron, alpha ✓

—— buffer penuh – circular overwrite ———————
[ADD]  Stack & Queue  Jan 16  ⏳ pending  ← timpa slot lama
head=1  tail=1  count=10/10  ⚠️ buffer penuh
```

---

## 💡 Keputusan Desain

### Mengapa multi-linked dan bukan `dict { tag: [list] }`?
Dict biasa menduplikasi referensi dan tidak ada hubungan langsung antar note dalam satu tag. Multi-linked menjaga satu node tunggal yang berpartisipasi di banyak chain — sesuai prinsip Section 9.3.

### Mengapa dua chain sorted dan bukan sort saat query?
Sort saat query = O(n log n) setiap kali tampilan dibuka. Chain yang dijaga terurut saat insert membuat view selalu O(n) traversal sederhana.

### Mengapa circular buffer dan bukan linked list untuk sync?
Sync hanya butuh N perubahan terakhir. Circular buffer menjamin memori konstan O(BUFFER_SIZE) dan push O(1) tanpa alokasi baru — penting untuk aplikasi yang berjalan lama di perangkat mobile.

---

## 📚 Referensi

- Rance D. Necaise — *Data Structures and Algorithms Using Python*, Chapter 9
- Section 9.1 Doubly Linked List · Section 9.2 Circular Linked List · Section 9.3 Multi-Linked Lists
