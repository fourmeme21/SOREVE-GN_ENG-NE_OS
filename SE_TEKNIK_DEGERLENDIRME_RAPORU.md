# SOVEREIGN ENGINE OS v2.0
## Teknik Değerlendirme ve Güçlendirme Raporu

> **Rapor Amacı:** Sistemin hayata geçişini engelleyen veya yavaşlatan zayıf noktaları tespit etmek, her birine somut ve uygulanabilir çözüm üretmek.
> **Değerlendirme Ekseni:** Teknik doğruluk · Kullanıcı deneyimi · Pratikte uygulanabilirlik
> **Tarih:** 2026-04-30 | Temel belge: SOVEREIGN_ENGINE_OS_v2.0

---

## GENEL BULGULAR ÖZETİ

Sistemin mimarisi sağlam ve vizyonu tutarlı. Ancak **"tasarımdan ürüne geçiş"** köprüsünde 15 kritik boşluk var. Bu boşlukların büyük çoğunluğu teknik eksikten değil, **"nasıl başlanır"** ve **"bir şey bozulursa ne olur"** sorularının yanıtsız kalmasından kaynaklanıyor.

```
KATEGORİ               SAYI   ETKİ
───────────────────────────────────────────────
KRİTİK (hayata geçişi engeller)    5    🔴
YÜKSEK (UX/pratik bozulma)        6    🟠
ORTA (iyileştirme gerektiren)      4    🟡
───────────────────────────────────────────────
TOPLAM                            15
```

---

# KISIM I — KRİTİK ZAYIFLIKLAR

---

## ZD-01 · Rust Core Bootstrap Problemi
**Seviye:** 🔴 KRİTİK | **Etki:** Sistem hiç başlatılamaz

### Tespit Edilen Sorun

Belge, Rust binary'nin donanım parmak iziyle kilitlendiğini ve lisans doğrulaması gerektirdiğini açıklıyor. Ancak şu soruları yanıtsız bırakıyor:

- Kullanıcı **ilk lisansı** nasıl alacak?
- Parmak izi nasıl **sahibe gönderilecek**?
- Lisans dosyası binary'e nasıl **inject edilecek**?
- Güncelleme sunucusu **nerede barındırılacak**?

Mevcut belgede tek cümle: *"Lisans doğrulandıktan sonra çağrılır."* Bu yeterli değil. Bootstrapsız bir sistem çalışamaz.

### Çözüm

**İki aşamalı lisanslama akışı:**

```
KURULUM AKIŞI (Day 0):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ADIM 1 — Parmak izi üretimi (kullanıcı tarafında)
  $ sovereign-core init
  → CPU ID + MAC + Disk serial → SHA-256 hash
  → Çıktı: fingerprint.json (48 karakter hex)
  → "Bu kodu sahibe gönderin: a3f8b2c1..."

ADIM 2 — Lisans üretimi (sahip tarafında)
  $ sovereign-license issue --fingerprint a3f8b2c1... --expires 2027-01-01
  → ed25519 private key ile imzalanır
  → LICENSE.sig dosyası üretilir
  → Kullanıcıya e-posta / güvenli kanal ile gönderilir

ADIM 3 — Aktivasyon (kullanıcı tarafında)
  $ sovereign-core activate LICENSE.sig
  → İmza doğrulandı ✅
  → Parmak izi eşleşti ✅
  → Sistem aktif

ADIM 4 — Doğrulama
  $ sovereign-core status
  → Lisans: GEÇERLİ (2026-04-30 → 2027-01-01)
  → Donanım: BAĞLI
  → Core versiyon: v1.0.0
```

**Güncelleme sunucusu için minimum altyapı:**

```
Seçenek A (Basit — önerilen başlangıç):
  GitHub Releases → imzalı binary + update.manifest
  sovereign-core update komutu manifest'i kontrol eder

Seçenek B (Gelişmiş):
  Özel update server (Cloudflare Workers, maliyet ~$0)
  sovereign-core update --channel stable|beta
```

**Bu akışa eklenmesi gereken belge bölümleri:**
- `SOVEREIGN_CORE_INSTALL.md` — adım adım kurulum rehberi
- `sovereign-license` CLI aracı tasarımı
- Lisans yenileme ve donanım değişikliği prosedürü

---

## ZD-02 · Session Log Büyüme Krizi
**Seviye:** 🔴 KRİTİK | **Etki:** 100+ session sonra sistem yavaşlar, 200+ session'da çalışamaz hale gelir

### Tespit Edilen Sorun

`session_log.md` her session'da büyür. Kobrabet 125 session'da bu dosyayı zaten binlerce satıra ulaştırdı. Belge bunu fark ediyor ama çözüm sunmuyor. Pratik problem:

- AI bağlamına sığan **maksimum token** sınırlıdır (~200k token)
- 200. session'da `session_log.md` bu sınırı aşar
- Sistem **kendi hafızasını okuyamaz** hale gelir

### Çözüm

**Üç katmanlı hafıza mimarisi:**

```
KATMAN 1 — HOT (Son 5 session, her zaman yüklü)
  session_log_hot.md
  → Max 500 satır
  → Her session başında AI'a verilir
  → 5 session sonra WARM'a arşivlenir

KATMAN 2 — WARM (6-50 session arası, talep üzerine)
  session_log_warm_[yil]_[ay].md
  → Aylık arşiv dosyaları
  → AI "Geçen ay ne yaptık?" sorunca ilgili dosya istenir
  → Otomatik bölümleme ile topic'e göre erişim

KATMAN 3 — COLD (50+ session, özet formatında)
  session_log_cold_summary.md
  → Tüm geçmişin 200 satırlık özeti
  → Kritik kararlar + öğrenilen hatalar
  → Her 50 session'da AI tarafından sıkıştırılır
```

**Arşivleme komutları (The Bridge'e entegre edilecek):**

```typescript
// bridge/session-archiver.ts
interface ArchiveConfig {
  hot_max_sessions: 5;
  warm_retention_months: 6;
  cold_summary_trigger: 50;  // kaç session'da bir sıkıştır
}

async function archiveSession(
  currentLog: string,
  sessionNumber: number
): Promise<void> {
  if (sessionNumber % 5 === 0) {
    // HOT → WARM rotasyonu
    await rotateHotToWarm(currentLog);
  }
  if (sessionNumber % 50 === 0) {
    // AI ile COLD özet üret
    await generateColdSummary();
  }
}
```

**CORE.md güncelleme gereksinimi:**
```
// Mevcut:
"session_log.md oku"

// Olması gereken:
"session_log_hot.md oku (son 5 session)
 Daha eski context gerekirse:
   session_log_warm_[tarih].md iste
   session_log_cold_summary.md iste"
```

---

## ZD-03 · The Bridge Kısmi Başarısızlık Senaryosu
**Seviye:** 🔴 KRİTİK | **Etki:** Veri bütünlüğü bozulur, sistem tutarsız durumda kalır

### Tespit Edilen Sorun

The Bridge 5 dosyayı güncellerken:
1. Dosya 1 güncellendi ✅
2. Dosya 2 güncellendi ✅
3. Dosya 3 güncellendi ✅
4. **Sistem çöktü** ❌
5. Dosya 4 ve 5 güncellenmedi

Sonuç: Sistem yarı güncellenmiş, tutarsız durumda. Belge bu senaryoyu tamamen göz ardı ediyor.

### Çözüm

**Atomik dosya işlem protokolü:**

```typescript
// bridge/atomic-executor.ts

interface BundleTransaction {
  bundle_id: string;
  files: FileOperation[];
  checkpoint_path: string;  // /tmp/se_checkpoint_[bundle_id].json
}

async function executeAtomically(tx: BundleTransaction): Promise<ExecutionResult> {

  // ADIM 1: Checkpoint oluştur (başlamadan önce)
  const checkpoint = {
    bundle_id: tx.bundle_id,
    started_at: Date.now(),
    completed_files: [] as string[],
    status: "IN_PROGRESS"
  };
  await writeCheckpoint(tx.checkpoint_path, checkpoint);

  // ADIM 2: Her dosyayı işlemeden önce YEDEK al
  const backups = new Map<string, string>();
  for (const op of tx.files) {
    if (op.operation !== "CREATE" && await fileExists(op.path)) {
      backups.set(op.path, await readFile(op.path));
    }
  }

  // ADIM 3: İşlemleri uygula, her adımda checkpoint güncelle
  try {
    for (const op of tx.files) {
      await applyFileOperation(op);
      checkpoint.completed_files.push(op.path);
      await writeCheckpoint(tx.checkpoint_path, checkpoint);
    }
    checkpoint.status = "COMPLETED";
    await writeCheckpoint(tx.checkpoint_path, checkpoint);
    return { success: true };

  } catch (err) {
    // ADIM 4: Hata → Tüm değişiklikleri geri al
    console.error(`Bridge hatası: ${err}. Rollback başlatılıyor...`);
    for (const [path, content] of backups) {
      await writeFile(path, content);
    }
    // CREATE işlemlerini sil
    for (const op of tx.files) {
      if (op.operation === "CREATE" && checkpoint.completed_files.includes(op.path)) {
        await deleteFile(op.path);
      }
    }
    checkpoint.status = "ROLLED_BACK";
    await writeCheckpoint(tx.checkpoint_path, checkpoint);
    return { success: false, error: String(err), rolled_back: true };
  }
}

// Kilitli kalmış checkpoint'leri temizle (sistem yeniden başlarken)
async function resumeOrCleanCheckpoints(): Promise<void> {
  const checkpoints = await listCheckpoints("/tmp/se_checkpoint_*.json");
  for (const cp of checkpoints) {
    if (cp.status === "IN_PROGRESS") {
      console.warn(`Yarım kalmış işlem bulundu: ${cp.bundle_id}. Rollback uygulanıyor.`);
      await rollbackCheckpoint(cp);
    }
  }
}
```

---

## ZD-04 · Politika Çakışma Çözümü Yok
**Seviye:** 🔴 KRİTİK | **Etki:** Belirsiz davranış, güvenlik açığı riski

### Tespit Edilen Sorun

`evaluatePolicy()` fonksiyonu politikaları sırayla döner ve ilk DENY/ASK_HUMAN'ı döndürür. Bu yaklaşım iki sorunu beraberinde getiriyor:

1. **Sıra bağımlılığı:** P3 (finansal limit) ile P6 (kritik → insan onayı) çakışırsa hangisi kazanır?
2. **Sessiz çakışma:** İki politika birbirini iptal edebilir, ama hiçbiri tetiklenmez.

Örnek kriz senaryosu:
```
P3: stake > available_balance → DENY (insufficient funds)
P6: risk_profile = CRITICAL   → ASK_HUMAN

Aynı anda her ikisi de tetiklenir.
Hangisi döner? DENY mi ASK_HUMAN mi?
Mevcut kod: hangisi önce DENY/ASK_HUMAN dönerse o.
Bu tanımsız davranış → güvenlik açığı.
```

### Çözüm

**Öncelikli politika değerlendirme sistemi:**

```typescript
// policy/priority-evaluator.ts

// Politikalara öncelik seviyesi atanır
interface PrioritizedPolicy {
  name: string;
  priority: number;  // Düşük = daha önce değerlendir
  fn: (d: Decision) => PolicyResult | null;
}

const POLICY_REGISTRY: PrioritizedPolicy[] = [
  // Öncelik 1: Mutlak güvenlik (asla geçilemez)
  { priority: 1, name: "immutable_state",        fn: immutableStatePolicy },
  { priority: 1, name: "client_direct_forbidden", fn: clientDirectPolicy },

  // Öncelik 2: Kimlik ve yetki
  { priority: 2, name: "financial_role_check",   fn: financialRolePolicy },
  { priority: 2, name: "hierarchy_direction",    fn: hierarchyPolicy },

  // Öncelik 3: Finansal limitler
  { priority: 3, name: "financial_limit",        fn: financialLimitPolicy },
  { priority: 3, name: "non_negative_amount",    fn: nonNegativePolicy },

  // Öncelik 4: İnsan onayı (her şey geçerse)
  { priority: 4, name: "critical_requires_human", fn: criticalHumanPolicy },
];

// Sıralı ve öncelikli değerlendirme
export function evaluatePrioritized(decision: Decision): PolicyResult {
  const sorted = [...POLICY_REGISTRY].sort((a, b) => a.priority - b.priority);

  for (const policy of sorted) {
    const result = policy.fn(decision);
    if (!result) continue;

    // DENY her zaman ASK_HUMAN'dan önce gelir (güvenlik tarafında)
    if (result.decision === "DENY") {
      logPolicyDecision(policy.name, "DENY", result.error_code);
      return result;
    }
    if (result.decision === "ASK_HUMAN") {
      // Daha yüksek öncelikli DENY olmadığını kontrol et
      const higherDeny = sorted
        .filter(p => p.priority < policy.priority)
        .some(p => p.fn(decision)?.decision === "DENY");

      if (!higherDeny) {
        logPolicyDecision(policy.name, "ASK_HUMAN", result.error_code);
        return result;
      }
    }
  }

  return { decision: "ALLOW" };
}
```

**CORE.md'ye eklenecek kural:**
```
POLİTİKA ÖNCELİK KURALI:
Öncelik 1 (Mutlak) > Öncelik 2 (Kimlik) > Öncelik 3 (Limit) > Öncelik 4 (İnsan)
DENY her zaman ASK_HUMAN'ı geçer.
```

---

## ZD-05 · AI Protokol Uyum Garantisi Yok
**Seviye:** 🔴 KRİTİK | **Etki:** Uzun oturumlarda sistem disiplini çöker

### Tespit Edilen Sorun

CORE.md ve AI_AGENT.md AI'ya kuralları söylüyor. Ama:
- Uzun context'te AI protokolü **unutabilir**
- AI sıkıştırıldığında self-check'i atlayabilir
- Hiçbir **mekanik zorlama** mekanizması yok

Bu, sistemin en zayıf halkasıdır: Kurallar var, ama uygulanmasını zorlayan altyapı yok.

### Çözüm

**Üç kademeli uyum mekanizması:**

**Kademe 1 — Yapısal zorlama (Decision Object olmadan çalışmaz):**
```typescript
// bridge/compliance-gate.ts

// The Bridge bir çıktıyı işlemeden önce Decision Object arar
function requiresDecisionObject(aiOutput: string): ComplianceResult {
  const decisionPattern = /\[KARAR BİLDİRİMİ\][\s\S]*?intent:[\s\S]*?confidence:/;
  if (!decisionPattern.test(aiOutput)) {
    return {
      compliant: false,
      error: "MISSING_DECISION_OBJECT",
      message: "Bu çıktı işlenemiyor. AI'dan önce [KARAR BİLDİRİMİ] formatında niyet beyanı gerekiyor."
    };
  }
  return { compliant: true };
}
```

**Kademe 2 — Otomatik re-injection (her 15 mesajda):**
```
// CORE.md'ye eklenecek kural:
RE-INJECTION PROTOKOLÜ:
Her 15 mesajda bir Claude şunu söyler:
"Context koruması: Bu session'da [N] mesaj geçti.
 Protokol hatırlatması: Her öneri [KARAR BİLDİRİMİ] ile başlamalı.
 Devam edecek misiniz?"

// Bu zorunlu re-injection AI'nın protokolu hatırlamasını sağlar.
```

**Kademe 3 — Compliance rate takibi:**
```typescript
// Kaç mesajda Decision Object var, kaçında yok?
interface ComplianceMetrics {
  total_outputs: number;
  compliant_outputs: number;
  compliance_rate: number;   // < 80% → uyarı
  last_violation: string;    // Son ihlal zamanı
}

// Dashboard'da gösterilir: "AI Uyum Oranı: 94.2%"
// Düşerse: "Uyarı: AI protokolü atlıyor. Re-inject önerilir."
```

---

# KISIM II — YÜKSEK ÖNCELİKLİ ZAYIFLIKLAR

---

## ZD-06 · Onboarding Sürtünmesi — "İlk 48 Saat" Rehberi Yok
**Seviye:** 🟠 YÜKSEK | **Etki:** Yeni kullanıcı nereden başlayacağını bilemiyor

### Tespit Edilen Sorun

Belge sistemi detaylı açıklıyor ama şunu açıklamıyor: **"Bugün sabah 9'da sistemi indirdiysem, akşam 6'ya kadar nerede olacağım?"**

Bootstrap protokolü var (Bölüm 8.1) ama somut zaman tahmini ve kontrol noktaları yok.

### Çözüm

**İlk 48 Saat Planı (Yeni Proje):**

```
GÜN 1 — SABAH (2 saat): Temel Kurulum
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
09:00 — sovereign-core init → parmak izi al
09:15 — Lisans al ve activate et
09:30 — Template kopyala: cp -r se-template/ [proje]-os/
10:00 — CORE.md düzenle (proje adı, stack, roller)
10:30 — AI'ya boot komutu ver → ilk sağlık kontrolü
11:00 — Beklenen çıktı: "Sağlık: ✅ Temiz. Hazırım."

GÜN 1 — ÖĞLEDEN SONRA (3 saat): FAZ 0
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
13:00 — AI ile DB şema tasarımı başlar
14:30 — İlk migration dosyası yazılır (YYYYMMDDHHMMSS_init.sql)
15:30 — Supabase'de çalıştırılır
16:00 — test_matrix.md'ye ilk satır eklenir
17:00 — İlk session kapanış protokolü çalıştırılır
17:30 — Beklenen çıktı: 4 dosya güncellendi, session_log #1 yazıldı

GÜN 2 — SABAH (3 saat): FAZ 1 başlangıcı
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
09:00 — session_log_hot.md okunur, sağlık kontrolü
09:30 — İlk domain adapter yazılır
11:00 — İlk EXECUTE_ACTION kararı Decision Object ile test edilir
12:00 — Bridge üzerinden ilk dosya uygulaması yapılır
12:30 — Beklenen çıktı: Sistem çalışıyor, ilk özellik production'a gidiyor
```

**Doğrulama kriterleri (Day 2 sonu):**
```
✅ sovereign-core status → GEÇERLİ
✅ session_log_hot.md → en az 2 session
✅ MIGRATIONS/  → en az 1 dosya
✅ test_matrix.md → en az 3 test senaryosu
✅ Bridge → en az 1 başarılı uygulama
```

---

## ZD-07 · Dokümantasyon Kayması — Otomasyonsuz Sürdürülemez
**Seviye:** 🟠 YÜKSEK | **Etki:** 30+ session sonra ARCHITECTURE.md gerçeği yansıtmıyor

### Tespit Edilen Sorun

Sistem şunu söylüyor: *"ARCHITECTURE.md her DB değişikliğinde güncellenmeli."*
Ancak bunu kim doğrulayacak? AI mı? Manuel mi?

Kobrabet'te bile bu sorun yaşandı: `session_index.md` açık sorunlar listesinde *"ARCHITECTURE.md güncellenmedi"* görüldü.

### Çözüm

**Dokümantasyon Drift Dedektörü:**

```typescript
// bridge/doc-drift-detector.ts
// Her session kapanışında çalışır

async function detectDocDrift(db: SupabaseClient): Promise<DriftReport> {
  const drifts: string[] = [];

  // 1. DB tabloları ile ARCHITECTURE.md'yi karşılaştır
  const dbTables = await db.rpc("get_table_names").then(r => r.data);
  const archContent = await readFile("ARCHITECTURE.md");
  for (const table of dbTables) {
    if (!archContent.includes(table)) {
      drifts.push(`⚠️ Tablo "${table}" ARCHITECTURE.md'de yok`);
    }
  }

  // 2. MIGRATIONS/ ile ARCHITECTURE.md migration bölümünü karşılaştır
  const migrationFiles = await listFiles("MIGRATIONS/*.sql");
  for (const mig of migrationFiles) {
    const basename = path.basename(mig);
    if (!archContent.includes(basename)) {
      drifts.push(`⚠️ Migration "${basename}" ARCHITECTURE.md'ye eklenmemiş`);
    }
  }

  // 3. DEPENDENCIES.md ile gerçek import'ları karşılaştır
  // (basit regex ile dosyalardaki import'lar taranabilir)

  return {
    has_drift: drifts.length > 0,
    issues: drifts,
    severity: drifts.length > 3 ? "HIGH" : "MEDIUM"
  };
}

// Session kapanışına entegre edilir:
// "DriftReport: 2 uyuşmazlık bulundu. ARCHITECTURE.md güncellenmeden session kapanmaz."
```

---

## ZD-08 · AI Model Kayması — Versiyon Sabitleme Stratejisi Yok
**Seviye:** 🟠 YÜKSEK | **Etki:** Claude/GPT güncellenmesi CAR parsing'i kırar

### Tespit Edilen Sorun

Decision Object yapısı AI'nın tutarlı formatta çıktı üretmesine bağlı. Claude, GPT veya Gemini'nin yeni bir versiyonu çıktığında format değişebilir. Belgedeki `metadata.model: string` alanı bu riski kayıt altına alıyor ama **önlemiyor**.

### Çözüm

**Model Uyum Katmanı:**

```typescript
// decision/model-adapter.ts

// Her model versiyonu için bir normalizer
const MODEL_NORMALIZERS: Record<string, (raw: string) => Partial<Decision>> = {
  "claude-sonnet-4-6": parseClaudeFormat,
  "claude-opus-4-6": parseClaudeFormat,
  "gpt-4o": parseOpenAIFormat,
  "gemini-2.0-flash": parseGeminiFormat,
  "fallback": parseFallbackFormat,
};

function normalizeDecision(raw: string, model: string): Decision {
  const normalizer = MODEL_NORMALIZERS[model] ?? MODEL_NORMALIZERS["fallback"];
  const partial = normalizer(raw);

  // Eksik alanları varsayılan değerlerle tamamla
  return {
    id: partial.id ?? crypto.randomUUID(),
    intent: partial.intent ?? "EXECUTE_ACTION",
    status: "PENDING",
    ...partial
  };
}

// CORE.md'ye eklenecek kural:
// "Her session başında kullandığın model versiyonunu belirt.
//  Model değişikliğinde normalizer uyumluluğunu test et."
```

**Belge'ye eklenecek model versiyon politikası:**
```
MODEL VERSİYON YÖNETİMİ:
• Her proje için bir "sabit model" belirlenir (CORE.md'de)
• Model değişikliği → önce Shadow Mode'da test et
• session_log'a model değişikliği 🔄 sembolüyle işaret edilir
• Yeni model için normalizer yazılmadan değişiklik onaylanmaz
```

---

## ZD-09 · Çok Operatörlü Senaryo Tanımsız
**Seviye:** 🟠 YÜKSEK | **Etki:** İkinci geliştirici katıldığında sistem çalışmaz

### Tespit Edilen Sorun

Tüm sistem tek operatör (Dics + AI) modelini varsayıyor. `session_log.md`, `session_index.md`, politikalar tek bir bakış açısını yansıtıyor. İkinci bir geliştirici katıldığında:

- Hangi `session_log` doğru?
- Kim `failure_patterns.md`'ye yazar?
- Çakışan kararlar nasıl çözülür?

### Çözüm

**Hafif çok-operatör protokolü:**

```
ÇOK OPERATÖR YAPISI:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

session_log.md        → Paylaşılan arşiv (Git ile yönetilir)
session_log_[isim].md → Kişisel session notu (operatör bazlı)

KURAL:
• session_index.md: tek kaynak, sadece "baş operatör" günceller
• session_log.md: merge ile birleştirilir, her operatör kendi bloğunu yazar
• failure_patterns.md: PR/review süreci, iki operatör onayı gerekir
• ARCHITECTURE.md: otomatik drift detector ile korunur (ZD-07 çözümü)

ÇAKIŞMA ÇÖZÜMÜ:
• İki operatör aynı anda farklı session açarsa:
  Kural: Finansal işlem içeren session önceliklidir
  Yöntem: session_index.md'de "aktif operatör" alanı eklenir
```

---

## ZD-10 · Hata Mesajları Son Kullanıcıya Sızıyor
**Seviye:** 🟠 YÜKSEK | **Etki:** Kötü UX + potansiyel güvenlik ihlali

### Tespit Edilen Sorun

Soft steer mesajları teknik: `"UNAUTHORIZED_HIERARCHY"`, `"CLIENT_DIRECT_FORBIDDEN"`, `"IMMUTABLE_STATE"`. Bunlar AI'ya yönlendirme içindir. Ancak belge bu mesajların son kullanıcıya gösterilip gösterilmeyeceğini ayrıştırmıyor.

Eğer bir `UNAUTHORIZED_HIERARCHY` mesajı son kullanıcıya görünürse hem kötü UX hem de sistem bilgisi sızması.

### Çözüm

**Üç katmanlı mesaj sistemi:**

```typescript
// policy/message-layers.ts

interface PolicyMessage {
  // Katman 1: Rust Core / AI içi (teknik, debug)
  technical_code: string;        // "UNAUTHORIZED_HIERARCHY"
  technical_detail: string;      // "Transfer direction: child_to_parent"

  // Katman 2: AI / Geliştirici mesajı (soft steer)
  developer_steer: string;       // "Transfer yalnızca parent→child yönünde çalışır."

  // Katman 3: Son kullanıcı mesajı (UX dostu)
  user_message: string;          // "Bu işlemi gerçekleştiremezsiniz."
  user_action?: string;          // "Yetkilendirilmiş bir işlem için destek alın."
}

const POLICY_MESSAGES: Record<string, PolicyMessage> = {
  UNAUTHORIZED_HIERARCHY: {
    technical_code: "UNAUTHORIZED_HIERARCHY",
    technical_detail: "Transfer direction invalid",
    developer_steer: "Transfer yalnızca parent→child yönünde çalışır. Hiyerarşiyi kontrol edin.",
    user_message: "Bu transfer işlemi gerçekleştirilemedi.",
    user_action: "Lütfen destek ekibiyle iletişime geçin."
  },
  INSUFFICIENT_FUNDS: {
    technical_code: "INSUFFICIENT_FUNDS",
    technical_detail: "balance + credit_limit < stake",
    developer_steer: `Kullanılabilir: ${0} TRY, Gerekli: ${0} TRY`,
    user_message: "Yetersiz bakiye.",
    user_action: "Bakiyenizi yükleyerek tekrar deneyin."
  }
  // ... tüm kodlar için
};

// Kullanım:
function getMessageForAudience(
  code: string,
  audience: "technical" | "developer" | "user"
): string {
  const msg = POLICY_MESSAGES[code];
  if (!msg) return audience === "user" ? "İşlem gerçekleştirilemedi." : `Unknown: ${code}`;
  switch (audience) {
    case "technical":  return msg.technical_detail;
    case "developer":  return msg.developer_steer;
    case "user":       return msg.user_action ? `${msg.user_message} ${msg.user_action}` : msg.user_message;
  }
}
```

---

## ZD-11 · Test Otomasyonu Eksik — Manuel Matrix Sürdürülemez
**Seviye:** 🟠 YÜKSEK | **Etki:** 50+ feature sonra test güncellemeleri gecikir

### Tespit Edilen Sorun

`test_matrix.md` manuel bir tablo. Her özellik için satır ekleniyor, durum ❓/✅/❌ işaretleniyor. Bu yaklaşım:
- 50+ özellikte takip zorlaşır
- Deploy öncesi tüm ❓ kontrolleri insan tarafından yapılmalı
- CI/CD ile entegrasyonu yok

### Çözüm

**test_matrix.md → Jest/Playwright otomasyonu köprüsü:**

```typescript
// tests/matrix-runner.ts
// test_matrix.md'yi parse eder ve otomatik test runner'a dönüştürür

import { parse } from "./matrix-parser";

const matrix = await parse("test_matrix.md");

describe("Sovereign Engine Test Matrix", () => {
  for (const section of matrix.sections) {
    describe(section.title, () => {
      for (const row of section.rows) {
        if (row.status === "❓") {
          // ❓ → otomatik test yazılmayı bekliyor
          test.todo(row.scenario);
        } else {
          test(row.scenario, async () => {
            // Her test satırı için otomatik test skelton
            // Geliştirici sadece body'yi doldurur
          });
        }
      }
    });
  }
});
```

**test_matrix.md format genişlemesi:**

```markdown
| # | Senaryo | Beklenen | Durum | Test Dosyası |
|---|---------|----------|-------|--------------|
| PB-01 | place_bet yeterli bakiye | 201 OK | ✅ | tests/bet/place-bet.test.ts:L12 |
| PB-02 | place_bet yetersiz bakiye | 400 INSUFFICIENT_FUNDS | ❓ | — |
```

---

# KISIM III — ORTA ÖNCELİKLİ ZAYIFLIKLAR

---

## ZD-12 · Token Bütçesi — Alarm Mekanizması Yok
**Seviye:** 🟡 ORTA | **Etki:** Maliyet kontrolsüz artar

### Tespit Edilen Sorun

`metadata.token_budget_spent` alanı var ama:
- Günlük/aylık **bütçe tavanı** tanımlanmamış
- Tavan aşılınca sistem **otomatik durdurulmuyor**
- Maliyet raporlama mekanizması yok

### Çözüm

```typescript
// budget/token-budget-manager.ts

interface BudgetConfig {
  daily_token_limit: number;    // Örn: 500_000
  session_token_limit: number;  // Örn: 50_000
  alert_threshold: number;      // %80'de uyarı
  hard_stop: boolean;           // %100'de durdur mu?
}

class TokenBudgetManager {
  async check(sessionTokens: number): Promise<BudgetCheckResult> {
    const dailyUsed = await this.getDailyUsage();
    const dailyRemaining = this.config.daily_token_limit - dailyUsed;

    if (dailyUsed >= this.config.daily_token_limit) {
      return {
        allowed: !this.config.hard_stop,
        warning: "DAILY_BUDGET_EXCEEDED",
        message: `Günlük token bütçesi aşıldı (${dailyUsed.toLocaleString()} / ${this.config.daily_token_limit.toLocaleString()})`
      };
    }

    if (dailyUsed / this.config.daily_token_limit >= this.config.alert_threshold) {
      return {
        allowed: true,
        warning: "BUDGET_ALERT",
        message: `Uyarı: Günlük bütçenin %${Math.round(dailyUsed/this.config.daily_token_limit*100)}'i kullanıldı`
      };
    }
    return { allowed: true };
  }
}
```

---

## ZD-13 · Dashboard Gerçek İmplementasyon Spec'i Yok
**Seviye:** 🟡 ORTA | **Etki:** Arayüz geliştirme başlayamıyor

### Tespit Edilen Sorun

Bölüm 7'deki beş ekran ASCII art ile tanımlanmış. Tauri + React seçilmiş ama:
- Hangi shadcn/ui bileşenleri kullanılacak?
- State yönetimi nasıl yapılandırılacak?
- Sovereign Core'dan gelen verinin formatı ne?

### Çözüm

**Minimum Viable Dashboard (MVD) — Component Listesi:**

```typescript
// ui/components/dashboard/

// 1. DecisionFeed.tsx — Canlı karar akışı
// Props: decisions: Decision[], onSelect: (id: string) => void
// Kullanılan: shadcn/ui Table + Badge + ScrollArea

// 2. PolicySummaryCard.tsx — Politika özeti
// Props: allow_rate: number, deny_rate: number, human_rate: number
// Kullanılan: shadcn/ui Card + Progress

// 3. SessionHealthPanel.tsx — Sağlık kontrolü
// Props: health_items: HealthItem[]
// Kullanılan: shadcn/ui Alert + CheckCircle icons

// 4. DecisionDetailDrawer.tsx — Karar detayı
// Props: decision: Decision | null, onClose: () => void
// Kullanılan: shadcn/ui Sheet + Tabs (5 katmanlı trace)

// 5. BridgeUploader.tsx — AI çıktısı yapıştır/uygula
// Props: onBundle: (bundle: SuperBundle) => void
// Kullanılan: shadcn/ui Textarea + Button + Alert

// Sovereign Core'dan veri çekme:
// import { evaluateDecision } from '../sovereign-core.node'
// Tauri command: invoke('get_audit_chain', { limit: 50 })
```

**State management şeması:**

```typescript
// ui/stores/dashboard-store.ts (Zustand)
interface DashboardStore {
  decisions: Decision[];
  selected_decision: Decision | null;
  health_status: HealthStatus;
  session_info: SessionInfo;
  compliance_rate: number;

  // Actions
  fetchDecisions: () => Promise<void>;
  selectDecision: (id: string) => void;
  runHealthCheck: () => Promise<void>;
}
```

---

## ZD-14 · Felaket Kurtarma Planı Eksik
**Seviye:** 🟡 ORTA | **Etki:** Donanım arızasında sistem kilitlenir

### Tespit Edilen Sorun

Donanım parmak iziyle kilitli Rust binary, şu senaryolarda çalışmayı durdurur:
- Geliştirme bilgisayarı bozulursa
- Disk değiştirilirse
- Sanal makine yeniden oluşturulursa

Belge bu senaryoları ele almıyor.

### Çözüm

```
FELAKET KURTARMA PLANI:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SENARYO 1: Donanım değişikliği (planlı)
  1. Eski donanımda: sovereign-core export-fingerprint
  2. Sahibine bildir: "Donanım değişiyorum, yeni parmak izi gelecek"
  3. Yeni donanımda: sovereign-core init → yeni parmak izi
  4. Sahibinden yeni lisans al → activate

SENARYO 2: Acil donanım arızası
  1. Sahibine e-posta: "Acil donanım arızası, eski fingerprint: [kayıtlı değer]"
  2. Sahip emergency lisans çıkarır (48 saat geçerli, kısıtlı)
  3. Yeni donanımda aktive edilir
  4. Normal lisans 7 gün içinde tamamlanır

ÖNLEM (Proaktif):
  Haftalık: sovereign-core backup-config → fingerprint + lisans yedeklenir
  Yedek dosya: encrypted USB'de saklanır
  Test: Ayda bir "disaster recovery drill" yapılır
```

---

## ZD-15 · Hard Lock Bug Senaryosu
**Seviye:** 🟡 ORTA | **Etki:** Sistemin bir parçası kalıcı olarak yanlış çalışır

### Tespit Edilen Sorun

*"Hard locks hiçbir zaman değiştirilemez"* — bu güvenlik için doğru ama şu soruyu doğuruyor: Hard lock'un kendisinde bir bug varsa ne olur?

Örnek: `immutable_state` hard lock'u, `"draft"` durumunu da yanlışlıkla locked sayıyor olsa, `"draft"` kayıtlar hiç güncellenemiyor. Ve bunu düzeltmek için binary'nin yeniden derlenmesi gerekiyor.

### Çözüm

**Hard lock kategorileri:**

```
GERÇEKTen HARD (hiç değiştirilemez):
  → Matematiksel güvenlik garantileri (hash chain bütünlüğü)
  → Cryptographic operasyonlar (imza doğrulama algoritması)

NEREDEYSE HARD (acil durumda sahip güncelleyebilir):
  → İş mantığı kuralları (won/lost listesi, rol kısıtlamaları)
  → Bu kurallar aslında "çok yüksek öncelikli soft rules"

YANLIŞLIKLA HARD YAPILAN:
  → Domain-spesifik listeler (locked_states = ["won", "lost"...])
  → Bunlar kullanıcı güncelleme kanalına taşınmalı
```

**Güncelleme politikası:**
```
HARD LOCK BUG PROTOKOLÜ:
1. Bug tespit edilir → sahibi bilgilendirilir
2. Sahip 24 saat içinde emergency patch binary üretir
3. İmzalanır ve güncelleme kanalına yüklenir
4. Kullanıcı: sovereign-core update --emergency
5. Yeni binary aktive edilir

SLA: Critical hard lock bug → 24 saat düzeltme garantisi
```

---

# KISIM IV — GÜÇLENDIRME YOL HARİTASI

---

## Öncelik Sıralaması ve Tahmini Yük

```
SPRINT 1 (Sistem çalışabilir hale gelir):
  ZD-01 Bootstrap    → 2 gün  [Rust CLI + lisans sunucusu]
  ZD-03 Bridge Atom  → 1 gün  [TypeScript rollback protokolü]
  ZD-04 Policy Order → 0.5 gün [Öncelik sistemi]
  ZD-06 Onboarding   → 0.5 gün [48 saat rehberi yazımı]
  ─────────────────────────────
  Toplam: ~4 gün

SPRINT 2 (Sistem güvenilir hale gelir):
  ZD-02 Log Arşiv    → 1 gün  [Hot/Warm/Cold katman]
  ZD-05 AI Uyum      → 1 gün  [Compliance gate + re-injection]
  ZD-07 Doc Drift    → 1 gün  [Drift dedektörü]
  ZD-10 Mesaj Katman → 0.5 gün [Kullanıcı/developer/teknik ayrımı]
  ─────────────────────────────
  Toplam: ~3.5 gün

SPRINT 3 (Sistem ölçeklenebilir hale gelir):
  ZD-08 Model Drift  → 1 gün  [Model normalizer]
  ZD-09 Multi-op     → 0.5 gün [Operatör protokolü]
  ZD-11 Test Otomasyon → 1 gün [Matrix runner]
  ZD-12 Bütçe Alarm  → 0.5 gün [Token manager]
  ZD-13 Dashboard    → 3 gün  [MVD component'lar]
  ZD-14 Felaket Plan → 0.5 gün [Dokümantasyon]
  ZD-15 Hard Lock    → 0.5 gün [Kategori ayrımı]
  ─────────────────────────────
  Toplam: ~7 gün
```

---

## Kritik Başarı Koşulları

Aşağıdaki 5 koşul sağlanmadan sistem hayata geçirilemez:

```
1. ✅ ZD-01 ÇÖZÜLDÜ: sovereign-core init → activate akışı çalışıyor
2. ✅ ZD-03 ÇÖZÜLDÜ: Bridge kısmi başarısızlıkta rollback yapıyor
3. ✅ ZD-04 ÇÖZÜLDÜ: Politika çakışmaları deterministic çözülüyor
4. ✅ ZD-02 ÇÖZÜLDÜ: Session log 200+ session sonra çökmüyor
5. ✅ ZD-06 ÇÖZÜLDÜ: Yeni kullanıcı 48 saatte sistemi çalıştırabiliyor
```

---

## Sistemin Gerçek Gücü (Değişmemesi Gereken)

Bu rapor zayıflıklara odaklandı. Sonuç olarak şunu belirtmek gerekir: Sistemin **felsefi çekirdeği sağlam**.

```
DEĞİŞMEMESİ GEREKEN:
  ✅ "AI karar önerir, sistem çalışıp çalışmayacağına karar verir"
  ✅ Fail-closed prensibi
  ✅ Hash zincirli imzalı audit log
  ✅ Soft steering (DENY değil yönlendirme)
  ✅ Domain-agnostik core + domain-spesifik adapter
  ✅ Session protokolü disiplini
  ✅ Pre-flight Read (bayat veri koruması)
  ✅ Kobrabet'ten gelen empirik hata kütüphanesi

GÜÇLENDİRİLMESİ GEREKEN:
  → Bootstrapsız kalma (ZD-01)
  → Hafıza yönetimi (ZD-02)
  → Atomik operasyonlar (ZD-03)
  → Deterministik politika sırası (ZD-04)
  → Mekanik protocol enforcement (ZD-05)
```

Sistemin büyük çoğunluğu doğru kurulmuş. Eksikler **inşa edilmemiş parçalar**, yanlış tasarlanmış parçalar değil.

---

*Sovereign Engine OS v2.0 — Teknik Değerlendirme Raporu*
*Hazırlanma: 2026-04-30 | Toplam zayıflık: 15 (5 Kritik · 6 Yüksek · 4 Orta)*
*Tahmini güçlendirme süresi: ~14.5 iş günü (3 sprint)*
