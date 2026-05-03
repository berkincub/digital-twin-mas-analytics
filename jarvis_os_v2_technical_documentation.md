# JARVIS Facility OS v2.0 - Teknik Dökümantasyon

## 1. Sisteme Genel Bakış

**JARVIS Facility OS**, karmaşık tesislerin (veri merkezleri, akıllı binalar, endüstriyel tesisler) otonom yönetimi ve güvenliği için tasarlanmış, **Çoklu Ajan Sistemine (Multi-Agent System - MAS)** dayalı yeni nesil bir Siber-Fiziksel Sistem (CPS) platformudur.

Sistem, geleneksel eşik tabanlı (heuristic) uyarı mekanizmalarının yol açtığı *"Yalancı Çoban" (False Alarm)* sendromunu aşmak için tamamen akademik, istatistiksel ve nedensel (causal) temeller üzerine inşa edilmiştir.

### Temel Özellikler:
- **Çoklu Ajan Mimarisi:** İletişimi izole edilmiş ayrı CPU process'leri üzerinde yürüyen asenkron ajanlar.
- **Akademik Kararlılık:** Lyapunov Kararlılık Teorisi ve Mahalanobis mesafesi ile zamansal sapmaların (temporal drift) tespiti.
- **Nedensel Analiz:** Judea Pearl'ün *do-calculus* metodolojisine dayalı karşıolgusal (counterfactual) kök neden analizi.
- **Stres Testi:** STPA (System-Theoretic Process Analysis) tabanlı sürekli Jacobian sızma simülasyonları.

---

## 2. Mimari ve Ajanlar (Multi-Agent System)

Sistem dört ana katmandan ve onları yöneten bir orkestratörden oluşur. Tüm bileşenler `mas_launch.py` tarafından yönetilir.

```mermaid
graph TD
    subgraph Orkestratör Katmanı
        O[ProcessSupervisor / mas_launch.py]
    end

    subgraph MAS İşlem Düğümleri
        A[Inference Engine / Defender]
        B[Semantic Governor / FRIDAY]
        C[Adversarial Simulator / MOCKER]
    end

    subgraph Kullanıcı ve Çıktı Katmanı
        D[APP / Cockpit GUI]
        E[(PowerBI CSV / DB)]
    end

    O -->|Restart/Supervise| A
    O -->|Restart/Supervise| B
    O -->|Restart/Supervise| C
    O -->|Launch| D
    
    A <-->|IPC Queues| B
    B -->|Structured Data| E
    B -->|Semantic Report| D
    A -->|Mitigation| D
    C -.->|STPA Signal (MQTT)| A
```

### 2.1. `inference_engine` (Defender / JARVIS)
Fiziksel sensörlerden gelen verileri dinleyen, BiRNN (Bidirectional Recurrent Neural Network) ve GCN (Graph Convolutional Network) tabanlı yapay zeka modelini çalıştıran savunma hattıdır. Kendi başına hızlı ve otonom tepki (`MitigationCmd`) verebilme yetkisine sahiptir.

### 2.2. `semantic_governor` (Governor / FRIDAY)
Sistemin beynidir. Defender'dan gelen anormallikleri bir Bilgi Grafiği (Knowledge Graph) ile eşleştirir, LLM tabanlı nedensel bir yorumlama yapar ve `SemanticReport` formatına dönüştürerek analiz edilmesini sağlar.

### 2.3. `adversarial_simulator` (Simulator / MOCKER)
"Red Team" (Kırmızı Takım) simülatörüdür. Sistemi sürekli olarak test etmek için arka planda sahte siber/fiziksel kriz senaryoları üretir. Savunma ajanını eğitmek için vazgeçilmez bir bileşendir.

### 2.4. `APP` (Cockpit GUI)
Asenkron mimariye sahip, `customtkinter` tabanlı izleme ve kontrol arayüzüdür. Tüm veri akışını gerçek zamanlı gösterir ve insanın (Human-in-the-loop) müdahalesine olanak tanır.

---

## 3. İletişim Protokolü (IPC - Inter-Process Communication)

Ajanlar kendi izole bellek alanlarında çalıştığı için (CPU izolasyonu), iletişim Python'un `multiprocessing.Queue` yapısı ve veri sınıfları (`dataclass`) kullanılarak sağlanır. Tüm protokol tanımları `shared/ipc_protocol.py` içerisindedir.

| Veri Paketi Sınıfı | Kaynak Ajan | Hedef Ajan / Katman | Açıklama |
| :--- | :--- | :--- | :--- |
| `AlertPacket` | Inference Engine | Semantic Governor | Sensörlerde tespit edilen yüksek frekanslı anormallik paketi (Skor, GCN embedding, Vektör). |
| `SemanticReport` | Semantic Governor | APP / PowerBI | İşlenmiş nedensel analiz, kök neden ve bilgi grafiği raporu. |
| `MitigationCmd` | Inference Engine | APP / Tesis | Otonom fiziksel eylem komutu (Örn: "Vana Kapat"). |
| `STPAScenarioSignal`| Adversarial Simulator| Inference Engine | MQTT üzerinden yayınlanan saldırı/anormallik simülasyonu paketi. |
| `UserCommand` | APP | Tüm Ajanlar | Kullanıcının arayüz üzerinden gönderdiği sistem komutları (Başlat/Durdur). |

> [!TIP]
> **Queue Yapısı:** `friday_queue` sadece AlertPacket'leri yüksek hızda taşırken (max 500 obje), `app_queue` daha büyük (max 2000 obje) kapasiteyle GUI'ye zengin veri aktarır.

---

## 4. İstatistiksel Denetim ve Kararlılık Boru Hattı

Sistem, sadece eşik geçişlerini değil, **zamanla artan kararsızlığı** ölçmek için gelişmiş matematiksel modeller kullanır. (`run_statistical_analysis.py` ve `test_statistical_audit.py` üzerinden denetlenir).

1. **Mahalanobis Mesafesi & Temporal Drift:**  
   Sensör verilerinin kendi normal dağılımından ne kadar saptığını ve bu sapmanın rastgele mi yoksa bir eğilim mi (drift) olduğunu tespit eder.

2. **Lyapunov Kararlılık Teorisi:**  
   Sistemin durum (state) denklemlerini analiz ederek, bir anormalliğin kontrol edilebilir mi (asimptotik olarak kararlı) yoksa kaosa doğru mu ilerlediğini hesaplar.

3. **Entropy-Regularized Ciddiyet (Severity) Ataması:**  
   Alarmların ciddiyetini (CRITICAL, HIGH, MEDIUM) entropiye göre ağırlıklandırır. Gürültülü ve düzensiz verilerde alarm fırtınalarını (False Alarm) keser.

> [!IMPORTANT]
> **Fizik Kilidi (Physics Lock):** Doğrudan fiziksel denklemlerin dışına çıkıldığı bariz bir şekilde tespit edildiğinde (Örn: Termal Kaçak), `physics_locked=True` bayrağı aktifleşir ve sistem istatistiksel hafifletmeye bakmaksızın anında savunma prosedürlerine geçer.

---

## 5. Dizin Yapısı ve Temel Dosyalar

*   `mas_launch.py`: MAS orkestratörünün giriş noktasıdır. Tüm process'leri başlatır ve çökme anında "Exponential Backoff" ile yeniden başlatır.
*   `run_statistical_analysis.py`: İstatistiksel hesaplama modüllerinin bağımsız olarak çalıştırılıp tesis verisinin test edilmesini sağlar.
*   `test_statistical_audit.py`: Sistemin matematiksel bütünlüğünü ölçen test süitidir. (Hedef minimum 10/12 başarılı test).
*   `shared/ipc_protocol.py`: Veri akışını düzenleyen kontratlar dosyasıdır.
*   `outputs/powerbi_feed.csv`: Semantic Governor tarafından oluşturulan zengin verilerin harici dashboard'lara aktarıldığı ana çıktıdır.

---

## 6. Sistemin Çalıştırılması

Sistemi tam teşekküllü MAS modunda çalıştırmak için:

```cmd
.\run_mas.bat
```
Veya doğrudan Python üzerinden:
```cmd
python mas_launch.py
```
**Parametreler:**
*   `--no-mocker`: Saldırgan simülatörünü kapatır (Salt savunma modu).
*   `--no-friday`: LLM/Nedensellik yöneticisini kapatır (Sadece ham alarmlar).
*   `--debug`: Detaylı terminal loglarını açar.
