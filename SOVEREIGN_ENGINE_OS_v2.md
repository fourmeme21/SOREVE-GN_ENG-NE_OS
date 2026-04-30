# SOVEREIGN ENGINE OS — v2.0
## Taşınabilir AI Yazılım Fabrikası İşletim Sistemi
### Runtime Governance Layer + Rust Çekirdeği + Kontrol Arayüzü

> **Kimlik:** Kobrabet OS (125 session, v6.58) × Meta-Framework v1.0 birleşimi  
> **Çekirdek Felsefesi:** "Yapay zeka karar önerir. Sistem bu kararın yaşayıp yaşamayacağına karar verir."  
> **Hedef:** Tek operatör + AI ile endüstriyel ölçekte, hatasız, tahmin edilebilir yazılım üretimi  
> **Çekirdek Dil:** Rust (derlenmemiş kaynak paylaşılmaz — sadece binary dağıtım)  
> **Versiyon:** 2.0 | Kalibrasyon: 2026-04-30

---

## OKUMA HARİTASI

```
Sistemi ilk kez anlıyorum          → Bölüm 0 + 1 + 2
Rust çekirdeğini anlamak istiyorum → Bölüm 3
Arayüzü görmek istiyorum           → Bölüm 4
Yeni projeye uygulamak istiyorum   → Bölüm 7 + 8
Tahmin çıkarmak istiyorum          → Bölüm 9
Güvenlik mimarisini anlamak        → Bölüm 5 + 6
```

---

# KISIM I — TEMEL MİMARİ

---

## BÖLÜM 0 — SİSTEMİN ÜÇ OMURGASI

Bu OS üç güçlü sistemin sentezi ve evrimidir:

```
KOBRABET OS (v6.58)          META-FRAMEWORK v1.0          SOVEREIGN ENGINE v2.0
────────────────────         ──────────────────────        ──────────────────────────
session_log.md          →    Decision Trace / Audit   →    [RUST] AuditChain (hash zincir)
failure_patterns.md     →    Policy Engine rules      →    [RUST] PolicyKernel (immutable)
CORE.md                 →    AI Constitution          →    [RUST] ConstitutionGuard
test_matrix.md          →    Validation scenarios     →    [TS] ValidationEngine
AI_AGENT.md self-check  →    Pre-exec Validation      →    [RUST] PreFlightGate
ARCHITECTURE.md         →    Contract Layer           →    [TS] DomainAdapter
DEPENDENCIES.md         →    Side-effect Map          →    [RUST] SideEffectTracer
```

**Evrimin özeti:**
- Kobrabet → ne yapılmaması gerektiğini 125 session'da empirik öğretti
- Meta-Framework → bu öğrenmeleri formal mimari dile çevirdi
- Sovereign Engine v2.0 → ikisini Rust çekirdeğine döküp kopyalanamaz hale getirdi

---

## BÖLÜM 1 — TAM MİMARİ HARİTASI

### 1.1 Beş Katmanlı Runtime Governance

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  KATMAN 0: AI AJAN KATMANI  (Claude / Gemini / GPT — değiştirilebilir)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         │ Doğal dil → niyet çıktısı
         ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  KATMAN 1: DECISION ENGINE  [TypeScript — kullanıcı güncelleyebilir]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CAR (Canonical Action Representation):
  Doğal dil → Standart JSON Decision nesnesi
  intent / category / payload / context / metadata
         │
         ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  KATMAN 2: VALIDATION ENGINE  [TypeScript — kullanıcı güncelleyebilir]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Şema Doğrulama (Zod/JSON Schema 2020-12)
  Tip Doğrulama (iş mantığı kuralları)
  Bağlam Doğrulama (oturum + staleness check)
  Pre-flight Read (bayat veri tespiti)
         │ PASS / REJECTED
         ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  KATMAN 3: POLICY KERNEL  [RUST — kopyalanamaz çekirdek]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ConstitutionGuard     → AI davranış anayasası
  PolicyEvaluator       → Rol + limit + kural değerlendirme
  SideEffectTracer      → Yan etki haritası kontrolü
  HardLockRegistry      → Değiştirilemez kural seti
         │ ALLOW / DENY / ASK_HUMAN
         ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  KATMAN 4: EXECUTION GATE  [RUST — kopyalanamaz çekirdek]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  IdempotencyGuard      → Mükerrer işlem engeli (Redis/in-memory)
  DurableCheckpoint     → Kaldığı yerden devam
  AuditChain            → Hash zincirli imzalı log
  FailClosedGuard       → Varsayılan ret prensibi
         │ PERMIT / BLOCK
         ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  KATMAN 5: DOMAIN ADAPTER  [TypeScript — kullanıcı yazar]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Generic intent → Domain-spesifik RPC / API çağrısı
  Tek domain bilgisi burada toplanır
  Core katmanlar domain-agnostik kalır
         │
         ▼
  [Gerçek Sistem: Supabase / PostgreSQL / REST API / gRPC]
```

### 1.2 Yönetişim Olgunluk Seviyeleri

| Seviye | Tanım | Karar Kaynağı | Yürütme | Odak |
|--------|-------|---------------|---------|------|
| L1 | YZ Aracı | İnsan | İnsan | İçerik Güvenliği |
| L2 | YZ İş Akışı | Statik zincir | Kodlanmış akış | Doğruluk + RAG |
| L3 | YZ Sistemi | Dinamik araç | Kısmi otonomi | API Güvenliği |
| L4 | YZ Kontrollü | Geniş otonomi | Gözetimli kod | Yetki Kapsamı |
| **L5** | **Sovereign Engine** | **Otonom karar** | **Runtime Governance** | **Eylem Yörüngesi** |

Sovereign Engine v2.0 **L5 seviyesini** hedefler. Bu seviyede sistem kararın *doğruluğunu* değil, kararın *yörüngesini* — kimlik, yetki, bütçe tüketimi, yan etki — bütünsel olarak izler.

### 1.3 Proje Dosya Yapısı

```
/[proje-adi]-os/
  ├── SOVEREIGN_ENGINE_OS.md     ← Bu dosya (AI boot file, her session başında verilir)
  ├── CORE.md                    ← Proje anayasası (AI sınırları + kimlik)
  ├── AI_AGENT.md                ← AI çalışma disiplini + self-check listesi
  ├── session_index.md           ← Anlık durum pusulası
  ├── session_log.md             ← Kümülatif hafıza + Decision Trace arşivi
  ├── ARCHITECTURE.md            ← DB şema + RPC kontratları (Contract Layer)
  ├── failure_patterns.md        ← Hata kütüphanesi (Policy Kernel kaynak verisi)
  ├── test_matrix.md             ← Validasyon senaryoları
  ├── DEPENDENCIES.md            ← Yan etki haritası
  ├── ROADMAP.md                 ← Ürün yol haritası
  ├── rollback.md                ← Geri dönüş protokolleri
  └── MIGRATIONS/                ← SQL dosyaları (YYYYMMDDHHMMSS_açıklama.sql)

/sovereign-core/                 ← RUST ÇEKİRDEĞİ (binary dağıtım, kaynak paylaşılmaz)
  ├── sovereign-core.exe/.bin    ← Platform binary
  ├── sovereign-core.wasm        ← Browser/edge integration
  ├── LICENSE.sig                ← Donanım imzalı lisans
  └── update.manifest            ← İmzalı güncelleme kanalı

/domain/                         ← KULLANICI YAZDIĞI KATMAN
  ├── [proje]/adapter.ts
  ├── [proje]/policies.ts
  └── [proje]/rules.ts

/ui/                             ← KONTROL ARAYÜZİ (Tauri/React)
  ├── dashboard/
  ├── decision-explorer/
  ├── policy-editor/
  └── session-manager/
```

---

## BÖLÜM 2 — DECISION ENGINE (Karar Motoru)

### 2.1 Canonical Action Representation (CAR)

Faramesh (2026) çalışmasından alınan bu kavram, semantik olarak aynı eylemin farklı sözdizimlerinden bağımsız olarak özdeş bir yapıya dönüştürülmesini zorunlu kılar. AI hangi modelden gelirse gelsin, çıktısı aynı standarda sokulur.

```typescript
// types/decision.ts — Değişmez kontrat

type Intent =
  | "READ_DATA"       // Risk: Düşük  — veri okuma, PII maskeleme gerekebilir
  | "WRITE_DATA"      // Risk: Orta   — yeni kayıt, şema kontrolü gerekli
  | "EXECUTE_ACTION"  // Risk: Yüksek — RPC/iş akışı, politika denetimi zorunlu
  | "TRIGGER_EVENT"   // Risk: Orta   — kuyruk/bildirim, rate limiting
  | "MODIFY_STATE";   // Risk: Kritik — config/durum, geri dönüş planı zorunlu

type RiskProfile = "LOW" | "MEDIUM" | "HIGH" | "CRITICAL";
type RoleType = "admin" | "superadmin" | "agent" | "player" | string;
type Confidence = "HIGH" | "MEDIUM" | "LOW";

interface Decision {
  // Kimlik
  id: string;                    // UUID v7 — zaman sıralı, idempotency temeli

  // Niyet
  intent: Intent;
  category: string;              // "FINANCIAL" | "AUTH" | "DATA" | domain-spesifik

  // Eylem
  payload: {
    action_name: string;         // "place_bet" | "transfer_balance" | domain-spesifik
    params: Record<string, unknown>;
    assumed_state?: Record<string, unknown>; // AI'nın varsaydığı mevcut durum
  };

  // Bağlam
  context: {
    user_id: string;
    session_id: string;
    user_role: RoleType;
    risk_profile: RiskProfile;
    hierarchy_path?: string[];   // ["admin", "superadmin", "agent", "player"]
  };

  // Üretim metaverisi
  metadata: {
    model: string;               // Hangi AI modeli üretti
    timestamp: string;           // ISO 8601
    session_number: number;      // OS session sayacı
    confidence: Confidence;
    self_check_passed: boolean;
    token_budget_spent?: number;
  };

  // Durum (Runtime tarafından doldurulur)
  status: "PENDING" | "VALIDATED" | "POLICY_APPROVED" |
          "EXECUTING" | "COMPLETED" | "REJECTED" | "BLOCKED";
  audit_hash?: string;           // Hash chain bağlantısı
}
```

### 2.2 Intent Risk Matrisi

| Intent | Yan Etki Riski | Gerekli Doğrulama | Kobrabet Örneği |
|--------|----------------|-------------------|-----------------|
| `READ_DATA` | Düşük (veri sızıntısı) | Erişim yetkisi, PII maskeleme | Bakiye sorgulama |
| `WRITE_DATA` | Orta | Şema uygunluğu, çakışma kontrolü | Maç ekleme |
| `EXECUTE_ACTION` | Yüksek | Politika denetimi, insan onayı | `place_bet` RPC |
| `TRIGGER_EVENT` | Düşük-Orta | Rate limiting | Bildirim gönderme |
| `MODIFY_STATE` | Kritik | Statik analiz, geri dönüş planı | Sistem konfigürasyonu |

### 2.3 AI'ya Verilecek Karar Üretim Talimatı

AI her kod önerisinden **önce** şu formatı üretir. Bu format olmadan Execution Gate devreye girmez:

```
[KARAR BİLDİRİMİ]
intent: EXECUTE_ACTION
category: FINANCIAL
action: place_bet
risk: HIGH
touches_balance: true
idempotency_required: true
self_check_passed: true
confidence: HIGH
```

---

## BÖLÜM 3 — VALIDATION ENGINE (Doğrulama Motoru)

### 3.1 Üç Aşamalı Doğrulama Protokolü

**Aşama 1 — Şema Doğrulaması**

JSON Schema 2020-12 standartları referans alınır. Node.js'te Zod, Python'da Pydantic kullanılır.

```typescript
// validation/schema-validator.ts
import { z } from "zod";

const DecisionSchema = z.object({
  id: z.string().uuid(),
  intent: z.enum(["READ_DATA", "WRITE_DATA", "EXECUTE_ACTION", "TRIGGER_EVENT", "MODIFY_STATE"]),
  category: z.string().min(1).max(50),
  payload: z.object({
    action_name: z.string().min(1),
    params: z.record(z.unknown()),
    assumed_state: z.record(z.unknown()).optional()
  }),
  context: z.object({
    user_id: z.string().uuid(),
    session_id: z.string(),
    user_role: z.string(),
    risk_profile: z.enum(["LOW", "MEDIUM", "HIGH", "CRITICAL"])
  }),
  metadata: z.object({
    model: z.string(),
    timestamp: z.string().datetime(),
    session_number: z.number().positive(),
    confidence: z.enum(["HIGH", "MEDIUM", "LOW"]),
    self_check_passed: z.boolean()
  })
});

export function validateSchema(raw: unknown): ValidationResult {
  const result = DecisionSchema.safeParse(raw);
  if (!result.success) {
    return {
      status: "REJECTED",
      reason: result.error.issues.map(i => `${i.path.join(".")}: ${i.message}`).join("; ")
    };
  }
  return { status: "PASS", data: result.data };
}
```

**Aşama 2 — İş Mantığı (Tip) Doğrulaması**

```typescript
// validation/business-validator.ts
export function validateBusinessRules(decision: Decision): ValidationResult {
  const errors: string[] = [];

  // Finansal işlemlerde tutar kontrolü
  if (decision.category === "FINANCIAL") {
    const amount = decision.payload.params.amount as number;
    const stake = decision.payload.params.stake as number;
    const value = amount ?? stake;
    if (value !== undefined && value <= 0) {
      errors.push("Finansal tutar pozitif olmalıdır");
    }
  }

  // Geçmiş tarih kontrolü
  if (decision.payload.params.date) {
    const date = new Date(decision.payload.params.date as string);
    if (date < new Date()) {
      errors.push("Tarih geçmişte olamaz");
    }
  }

  // Confidence-Self-check tutarlılığı
  if (decision.metadata.confidence === "HIGH" && !decision.metadata.self_check_passed) {
    errors.push("HIGH confidence için self_check_passed zorunludur");
  }

  return errors.length > 0
    ? { status: "REJECTED", reason: errors.join("; ") }
    : { status: "PASS" };
}
```

**Aşama 3 — Bağlam (Context) Doğrulaması + Pre-flight Read**

Faramesh'in "bağlam kayması" (context drift) tanımı: Ajan oturum başındaki verileri hafızasında tutar, gerçek dünya değişmiş olabilir.

```typescript
// validation/context-validator.ts
export async function validateContext(
  decision: Decision,
  db: SupabaseClient
): Promise<ValidationResult> {

  // 1. Oturum geçerliliği
  const session = await db.from("sessions")
    .select("expires_at, user_id")
    .eq("id", decision.context.session_id)
    .single();

  if (!session.data || new Date(session.data.expires_at) < new Date()) {
    return { status: "REJECTED", reason: "SESSION_EXPIRED" };
  }

  // 2. Pre-flight Read — Finansal işlemlerde bayat veri tespiti
  if (decision.category === "FINANCIAL" && decision.payload.assumed_state) {
    const freshState = await db.from("profiles")
      .select("balance, credit_limit, is_active")
      .eq("id", decision.context.user_id)
      .single();

    if (!freshState.data?.is_active) {
      return { status: "REJECTED", reason: "ACCOUNT_INACTIVE" };
    }

    const assumedBalance = decision.payload.assumed_state.balance as number;
    const freshBalance = freshState.data.balance;

    // Tolerans: 0.01 TL'den fazla sapma → RE_EVALUATE
    if (Math.abs(assumedBalance - freshBalance) > 0.01) {
      return {
        status: "REJECTED",
        reason: "RE_EVALUATE",
        stale_fields: ["balance"],
        current_state: {
          balance: freshBalance,
          credit_limit: freshState.data.credit_limit
        }
      };
    }
  }

  return { status: "PASS" };
}
```

### 3.2 Evrensel Self-Check Protokolü (AI'nın Çalıştırdığı)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRE-EXECUTION SELF-CHECK (Her çözümden önce)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

FİNANSAL KONTROLLER:
[ ] Bakiyeye/finansal veriye dokunan kod var mı?
    → EVET: intent=EXECUTE_ACTION, category=FINANCIAL, RPC zorunlu, atomik transaction

[ ] Race condition riski var mı? (eş zamanlı iki istek)
    → EVET: FOR UPDATE kilidi, idempotency_key

[ ] Aynı işlem iki kez çalışabilir mi? (double-click / retry)
    → EVET: idempotency_key = Hash(user_id + action + params + zaman_penceresi)

YETKİ KONTROLLERI:
[ ] Yetki kontrolü eksik mi? (RLS bypass riski)
    → EVET: Policy Engine'den geç, rol hiyerarşisi kontrolü ekle

[ ] Hiyerarşi yönü doğru mu? (parent→child mi?)
    → HAYIR: UNAUTHORIZED_HIERARCHY — engelle

[ ] Won/lost/cancelled durumundaki kayda dokunuluyor mu?
    → EVET: IMMUTABLE_STATE — kesinlikle engelle

VERİ GÜVENLİĞİ:
[ ] Client'dan gelen değer direkt DB'ye gidiyor mu?
    → EVET: YASAK — her zaman server-side RPC

[ ] Türkçe karakter içeren route oluşturuluyor mu?
    → EVET: YASAK — ASCII-only route zorunlu

BAĞLAM:
[ ] Bu değişikliğin yan etkileri neler? DEPENDENCIES.md kontrol edildi mi?
[ ] DB değişikliği var mı? MIGRATIONS/ + ARCHITECTURE.md güncellendi mi?
[ ] test_matrix.md'de bu özelliğin test senaryosu eklendi mi?
[ ] Context eksik mi? (dosya istenmedi)
    → EVET: HIGH confidence yasak — önce dosyayı iste

SONUÇ:
Tüm kontroller temiz → confidence: HIGH
Bir belirsizlik → confidence: MEDIUM, gerekçe belirt
Context eksik   → confidence: LOW, dosya iste
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## BÖLÜM 4 — POLICY KERNEL (Politika Çekirdeği)

Bu katman **Rust ile yazılmıştır ve derlenmiş binary olarak dağıtılır.** Kaynak kodu paylaşılmaz; ancak sahibi tarafından imzalı güncelleme kanalı üzerinden güncellenebilir.

### 4.1 Politika Türleri

| Politika Türü | Kontrol | Red Örneği |
|---------------|---------|------------|
| Araç Erişimi | Ajanın bu aracı kullanma yetkisi var mı? | `UNAUTHORIZED_TOOL` |
| Kaynak Erişimi | Ajan bu veri setine erişebilir mi? | `FORBIDDEN_DATA_SCOPE` |
| Komut Güvenliği | Parametreler güvenli aralıkta mı? | `OUT_OF_BOUNDS_VALUE` |
| Rol Hiyerarşisi | İşlem yetki zinciriyle uyumlu mu? | `UNAUTHORIZED_HIERARCHY` |
| Operasyonel | Sistem şu an bu işleme izin veriyor mu? | `MAINTENANCE_WINDOW_ACTIVE` |
| Durum Kilidi | Değiştirilemez kayda müdahale var mı? | `IMMUTABLE_STATE` |
| Finansal Limit | Tutar izin verilen sınırda mı? | `INSUFFICIENT_FUNDS` |
| Kaynak Bütçesi | Token/adım bütçesi aşılıyor mu? | `BUDGET_EXCEEDED` |

### 4.2 TypeScript Policy Interface (Kullanıcı Katmanı)

Core politikalar Rust'ta; ancak kullanıcı kendi domain kurallarını TypeScript'te ekler:

```typescript
// policy/policy-engine.ts

interface PolicyResult {
  decision: "ALLOW" | "DENY" | "ASK_HUMAN";
  error_code?: string;
  soft_steer?: string;   // Ret yerine yönlendirme mesajı
  requires_human?: boolean;
}

// Chimera (Akarlar, 2026) modelinden uyarlanan soft steering:
// Ret yerine "nereye gidebilirsin" bilgisi ver

const DOMAIN_POLICIES: Record<string, (d: Decision) => PolicyResult | null> = {

  // P1: Rol bazlı finansal işlem yetkisi
  financial_role_check: (d) => {
    if (d.category !== "FINANCIAL") return null;
    const ALLOWED_ROLES = ["player"];
    if (!ALLOWED_ROLES.includes(d.context.user_role)) {
      return {
        decision: "DENY",
        error_code: "UNAUTHORIZED_ROLE",
        soft_steer: `${d.context.user_role} rolü kupon oynayamaz. Bakiye transferi için transfer_balance kullanın.`
      };
    }
    return { decision: "ALLOW" };
  },

  // P2: Hiyerarşi yönü (KOBRABET'te kanıtlanan kritik kural)
  hierarchy_direction: (d) => {
    if (d.payload.action_name !== "transfer_balance") return null;
    if (d.payload.params.direction === "child_to_parent") {
      return {
        decision: "DENY",
        error_code: "UNAUTHORIZED_HIERARCHY",
        soft_steer: "Transfer yalnızca parent'tan child'a yapılabilir. Hiyerarşiyi kontrol edin."
      };
    }
    return { decision: "ALLOW" };
  },

  // P3: Finansal limit (balance + credit_limit >= stake)
  financial_limit: (d) => {
    if (d.category !== "FINANCIAL") return null;
    const stake = d.payload.params.stake as number;
    const available = d.payload.params.available_balance as number;
    if (stake !== undefined && available !== undefined && stake > available) {
      return {
        decision: "DENY",
        error_code: "INSUFFICIENT_FUNDS",
        soft_steer: `Yetersiz bakiye. Kullanılabilir: ${available} TRY, Gerekli: ${stake} TRY`
      };
    }
    return null;
  },

  // P4: Değiştirilemez durum koruması (KOBRABET: won/lost kupon)
  immutable_state: (d) => {
    const LOCKED = ["won", "lost", "cancelled", "settled", "archived"];
    const state = d.payload.params.current_status as string;
    if (LOCKED.includes(state) && d.intent !== "READ_DATA") {
      return {
        decision: "DENY",
        error_code: "IMMUTABLE_STATE",
        soft_steer: `"${state}" durumundaki kayıt değiştirilemez. Yalnızca okuma yapılabilir.`
      };
    }
    return null;
  },

  // P5: Client-direct yasağı
  client_direct_forbidden: (d) => {
    if (d.payload.params.source === "client_direct") {
      return {
        decision: "DENY",
        error_code: "CLIENT_DIRECT_FORBIDDEN",
        soft_steer: "Finansal işlemler client'dan direkt DB'ye yazılamaz. Server-side RPC kullanın."
      };
    }
    return null;
  },

  // P6: CRITICAL risk → insan onayı
  critical_requires_human: (d) => {
    if (d.context.risk_profile === "CRITICAL") {
      return {
        decision: "ASK_HUMAN",
        error_code: "HUMAN_APPROVAL_REQUIRED",
        requires_human: true,
        soft_steer: "Bu işlem kritik risk seviyesinde. Dashboard üzerinden manuel onay gerekiyor."
      };
    }
    return null;
  },

  // P7: Bakiye işleminde negatif tutar
  non_negative_amount: (d) => {
    if (d.category !== "FINANCIAL") return null;
    const amt = (d.payload.params.amount ?? d.payload.params.stake) as number;
    if (amt !== undefined && amt <= 0) {
      return {
        decision: "DENY",
        error_code: "OUT_OF_BOUNDS_VALUE",
        soft_steer: "İşlem tutarı sıfır veya negatif olamaz."
      };
    }
    return null;
  }
};

// Ana değerlendirici
export function evaluatePolicy(decision: Decision): PolicyResult {
  for (const [name, policy] of Object.entries(DOMAIN_POLICIES)) {
    const result = policy(decision);
    if (result && result.decision !== "ALLOW") {
      console.log(`[POLICY:${name}] → ${result.decision} (${result.error_code})`);
      return result;
    }
  }
  return { decision: "ALLOW" };
}

// Yeni domain kuralı ekleme (core değişmez)
export function registerPolicy(
  name: string,
  fn: (d: Decision) => PolicyResult | null
): void {
  DOMAIN_POLICIES[name] = fn;
}
```

### 4.3 Soft Steering Prensibi

Chimera mimarisinden alınan bu yaklaşım, Kobrabet'te de doğrulandı: Sadece "403 Forbidden" vermek ajanın aynı hatayı tekrarlamasına neden olur. Bunun yerine:

```
YANLIŞ:  "DENIED"
DOĞRU:   "DENIED: [sebep] — Bunun yerine [alternatif] kullanabilirsiniz."

Örnek:
"Kullanıcıyı silme yetkiniz yok.
 Ancak kullanıcıyı 'pasif' duruma alabilirsiniz: update_status aracını deneyin."
```

Soft steer başarı oranı her session'da ölçülür ve `session_log.md`'ye kaydedilir.

---

# KISIM II — RUST ÇEKİRDEĞİ TASARIM PLANI

---

## BÖLÜM 5 — SOVEREIGN CORE: RUST MİMARİSİ

### 5.1 Neden Rust?

| Gereksinim | Rust Karşılığı |
|------------|---------------|
| Kopyalanamaz binary | `build.rs` ile donanım parmak izi (hardware fingerprint) + imza |
| Sıfır GC pause | Memory safety garantisi — finansal işlemlerde gecikme yok |
| FFI uyumluluğu | Node.js (napi-rs), Python (PyO3), WASM (wasm-bindgen) |
| Hash chain log | `sha2` crate ile immutable audit trail |
| Yüksek throughput | Async Tokio runtime — policy değerlendirmesi <1ms |
| Güncelleme kanalı | İmzalı binary (ed25519) — sadece sahibi günceller |

### 5.2 Crate Yapısı (Kaynak Gizli, Tasarım Açık)

```
sovereign-core/
  ├── Cargo.toml
  ├── build.rs                    ← Donanım parmak izi + lisans imzası
  ├── src/
  │   ├── lib.rs                  ← Public API
  │   ├── kernel/
  │   │   ├── mod.rs
  │   │   ├── policy_kernel.rs    ← PolicyKernel — hard-coded rules
  │   │   ├── execution_gate.rs   ← IdempotencyGuard + FailClosed
  │   │   ├── audit_chain.rs      ← Hash zincirli imzalı log
  │   │   ├── constitution.rs     ← ConstitutionGuard — AI anayasası
  │   │   └── side_effect.rs      ← SideEffectTracer
  │   ├── license/
  │   │   ├── mod.rs
  │   │   ├── fingerprint.rs      ← Donanım parmak izi
  │   │   └── verifier.rs         ← Lisans doğrulama
  │   ├── ffi/
  │   │   ├── node_binding.rs     ← napi-rs (Node.js entegrasyonu)
  │   │   └── wasm_binding.rs     ← wasm-bindgen (browser/edge)
  │   └── proto/
  │       └── decision.proto      ← gRPC/Protobuf şeması
  └── tests/
      ├── policy_tests.rs
      └── audit_tests.rs
```

### 5.3 PolicyKernel — Rust Tasarımı

```rust
// src/kernel/policy_kernel.rs
// [TASARIM PLANI — Kaynak paylaşılmaz, sadece API dokumentasyonu]

use std::collections::HashMap;

/// Politika değerlendirme sonucu
#[derive(Debug, Clone)]
pub enum PolicyDecision {
    Allow,
    Deny { code: String, steer: String },
    AskHuman { reason: String },
}

/// Değiştirilemez kural seti — binary içinde hard-coded
/// Dışarıdan erişilemez, sadece evaluate() API'si kullanılabilir
pub struct PolicyKernel {
    hard_locks: Vec<HardLockRule>,      // Kesinlikle değiştirilemez kurallar
    soft_rules: Vec<SoftRule>,          // Güncellenebilir domain kuralları
    version: u32,                       // Kural seti versiyonu
}

impl PolicyKernel {
    /// Lisans doğrulandıktan sonra çağrılır
    pub fn new_verified(license: &VerifiedLicense) -> Result<Self, CoreError> {
        // Hard locks: won/lost kayıt koruması, negative amount,
        //             client-direct yasağı, hierarchy direction
        // Soft rules: domain-spesifik, güncelleme kanalından alınabilir
        todo!("Kaynak gizli — sadece binary dağıtım")
    }

    /// Ana değerlendirme fonksiyonu — <1ms garantili
    pub fn evaluate(&self, decision: &DecisionPayload) -> PolicyDecision {
        // 1. Hard lock'ları kontrol et (değiştirilemez)
        // 2. Soft rule'ları kontrol et (güncellenebilir)
        // 3. Varsayılan: DENY (fail-closed)
        todo!("Kaynak gizli")
    }

    /// Sahibi tarafından imzalı güncelleme paketi uygular
    pub fn apply_update(&mut self, update: &SignedPolicyUpdate) -> Result<(), CoreError> {
        // Sadece soft_rules güncellenir
        // Hard locks hiçbir zaman değiştirilemez
        // İmza doğrulaması: ed25519 (sahibinin public key'i build zamanında baked-in)
        todo!("Kaynak gizli")
    }
}
```

### 5.4 ExecutionGate — Rust Tasarımı

```rust
// src/kernel/execution_gate.rs
// [TASARIM PLANI]

use sha2::{Sha256, Digest};
use std::time::{SystemTime, UNIX_EPOCH};

/// Idempotency anahtarı üretimi
/// 5 saniyelik zaman penceresi — aynı işlem bu sürede tekrar gelirse engel
pub fn generate_idempotency_key(
    user_id: &str,
    action: &str,
    params_hash: &str,
) -> String {
    let window = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs() / 5;  // 5 saniyelik pencere

    let mut hasher = Sha256::new();
    hasher.update(format!("{}:{}:{}:{}", user_id, action, params_hash, window));
    format!("{:x}", hasher.finalize())
}

/// Fail-closed garantisi
/// OPA/policy engine timeout → DENY (asla izin verme)
pub struct FailClosedGuard {
    timeout_ms: u64,
}

impl FailClosedGuard {
    pub async fn evaluate_with_timeout<F, T>(
        &self,
        fut: F,
    ) -> Result<T, GateError>
    where
        F: std::future::Future<Output = T>,
    {
        // Timeout aşılırsa → GateError::Timeout → DENY
        // Policy servisi çöksrse → GateError::ServiceUnavailable → DENY
        // Asla "izin ver" değil
        todo!("Kaynak gizli")
    }
}
```

### 5.5 AuditChain — Hash Zincirli İmzalı Log

```rust
// src/kernel/audit_chain.rs
// [TASARIM PLANI]

/// Her log kaydı önceki kaydın hash'ini içerir
/// Geriye dönük manipülasyon teknik olarak imkansız
#[derive(Debug, Clone)]
pub struct AuditEntry {
    pub id: String,
    pub timestamp: i64,
    pub decision_id: String,
    pub intent: String,
    pub policy_result: String,       // "ALLOW" | "DENY:kod"
    pub gate_result: String,         // "PERMIT" | "BLOCK"
    pub prev_hash: String,           // Zincir bağlantısı
    pub entry_hash: String,          // Bu kaydın hash'i
    pub signature: String,           // ed25519 imzası
}

/// Beş katmanlı Decision Trace
/// Faramesh + Chimera akademik modellerinden uyarlanan yapı
#[derive(Debug)]
pub struct DecisionTrace {
    pub trigger: String,             // 1. Tetikleyici olay
    pub context_snapshot: String,   // 2. Karar anındaki bağlam (JSON)
    pub reasoning_chain: Vec<String>, // 3. Chain-of-thought adımları
    pub alternatives_considered: Vec<String>, // 4. Değerlendirilen alternatifler
    pub authority: String,           // 5. Kim izin verdi, hangi kural tetiklendi
}
```

### 5.6 Lisans ve Donanım Parmak İzi

```rust
// src/license/fingerprint.rs
// [TASARIM PLANI]

/// Donanım parmak izi — binary bu donanıma bağlı
/// Kaynak başka bir donanımda derlense de çalışmaz
pub struct HardwareFingerprint {
    cpu_id: String,
    mac_addr: String,
    disk_serial: String,
}

impl HardwareFingerprint {
    /// Mevcut donanımın parmak izini hesapla
    pub fn current() -> Self {
        todo!("Platform-spesifik donanım sorgulama")
    }

    /// İmzalı lisansla karşılaştır
    pub fn verify(&self, license: &SignedLicense) -> Result<VerifiedLicense, LicenseError> {
        // ed25519 imzası doğrula (sahibinin public key'i hard-coded)
        // Parmak izi eşleşmesi kontrol et
        // Geçerlilik süresi kontrol et
        todo!("Kaynak gizli")
    }
}

/// Güncelleme kanalı
/// Sahip yeni bir policy paketi imzalar → binary'e inject edilir
/// Kaynak kod olmadan bile kurallar güncellenebilir
pub struct UpdateChannel {
    pub channel_url: String,         // Sahibin güncelleme sunucusu
    pub public_key: [u8; 32],        // ed25519 public key (build-time baked-in)
}
```

### 5.7 Node.js Binding (napi-rs)

```rust
// src/ffi/node_binding.rs
// [TASARIM PLANI — kullanım API'si]

use napi_derive::napi;

/// Node.js'ten çağrılabilir tek fonksiyon
/// Tüm core mantığı bu çağrının arkasında gizli
#[napi]
pub fn evaluate_decision(decision_json: String) -> String {
    // 1. Lisans doğrula
    // 2. JSON parse et
    // 3. PolicyKernel.evaluate() çalıştır
    // 4. ExecutionGate.process() çalıştır
    // 5. AuditChain.append() ile kayıt oluştur
    // 6. Sonucu JSON olarak döndür
    todo!("Kaynak gizli")
}

// TypeScript kullanımı:
// import { evaluateDecision } from './sovereign-core.node'
// const result = evaluateDecision(JSON.stringify(decision))
// const { status, receipt, soft_steer } = JSON.parse(result)
```

---

## BÖLÜM 6 — GÜVENLİK MODELİ

### 6.1 OWASP Top 10 Tehdit Matrisi

| Tehdit | Açıklama | Sovereign Engine Koruması |
|--------|----------|-----------------------------|
| Prompt Injection | Dış girdinin AI talimatlarını değiştirmesi | PolicyKernel kararı modelden bağımsız değerlendirir |
| Data Exfiltration | Yetkisiz veri sızdırma | READ_DATA niyetinde PII maskeleme + çıkış filtresi |
| Runaway Agent | Kontrolsüz döngüye giren ajan | Step budget + zaman sınırı (ExecutionGate) |
| Shadow AI | Denetimsiz ajan kurulumu | ConstitutionGuard + Ajan Registry zorunluluğu |
| Replay Attack | Aynı eylemin tekrar gönderilmesi | IdempotencyGuard + AuditChain hash doğrulama |
| Privilege Escalation | Yetki aşımı girişimi | HardLockRegistry — değiştirilemez kural |
| Context Manipulation | Bayat veri ile karar aldırma | Pre-flight Read + RE_EVALUATE sinyali |
| Double Spend | Çift finansal işlem | FOR UPDATE kilidi + Idempotency key |

### 6.2 Fail-Closed Prensibi (Kritik)

```
KURAL: Şüpheli durumda varsayılan karar her zaman DENY'dır.

OPA/Policy timeout     → DENY (izin verme)
Policy servisi çökmesi → DENY (izin verme)
Eksik kural            → DENY (izin verme)
İmza hatası            → DENY (izin verme)
Bilinmeyen intent      → DENY (izin verme)

ASLA: "Servis erişilebilir değil, geçici olarak izin ver"
```

### 6.3 Sandboxing (Yüksek Riskli İşlemler)

MODIFY_STATE ve CRITICAL risk seviyesindeki işlemler için izole yürütme:

```
Geliştirme: Docker container isolasyonu
Prodüksiyon: gVisor (Google) veya Kata Containers
Gelecek: WASM sandbox (edge deployment için)
```

---

# KISIM III — KONTROL ARAYÜZİ

---

## BÖLÜM 7 — SOVEREIGN DASHBOARD (Arayüz Tasarım Planı)

### 7.1 Teknoloji Seçimi

```
Frontend Framework:  React + Vite (web) / Tauri (desktop binary)
UI Kütüphanesi:     Tailwind CSS + shadcn/ui
State Yönetimi:     Zustand
Real-time:          Supabase Realtime / WebSocket
Rust Entegrasyonu:  Tauri commands (desktop) / WASM (web)
```

**Neden Tauri?**
- Rust core ile native entegrasyon
- Web teknolojileriyle geliştirilir ama native binary çıkarır
- Electron'dan 10x daha hafif
- Kross-platform (Windows/macOS/Linux)

### 7.2 Beş Ana Ekran

#### EKRAN 1 — Ana Kontrol Paneli (Mission Control)

```
┌─────────────────────────────────────────────────────────────┐
│  SOVEREIGN ENGINE — CONTROL CENTER              v2.0  [●●●] │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  KARARLAR    │   POLİTİKA   │   OTURUMLAR  │   SİSTEM       │
│  Son 24s     │   İhlaller   │   Aktif      │   Sağlık       │
│  ████ 247    │   ▓▓ 3       │   ◉ 2        │   ✅ Temiz     │
├──────────────┴──────────────┴──────────────┴────────────────┤
│                                                             │
│  CANLI KARAR AKIŞI                          [Duraklat] [▶] │
│  ─────────────────────────────────────────────────────     │
│  12:34:01  place_bet     ALLOW    player/usr_4432  ✅       │
│  12:34:00  READ_DATA     ALLOW    agent/usr_1201   ✅       │
│  12:33:59  transfer_bal  DENY     agent→agent ⚠️  ❌        │
│            └─ UNAUTHORIZED_HIERARCHY                        │
│            └─ Steer: "Transfer yalnızca parent→child"      │
│  12:33:58  MODIFY_STATE  ASK_HUMAN  admin  🔔               │
│  ─────────────────────────────────────────────────────     │
│                                               [Tümünü Gör] │
├─────────────────────────────────────────────────────────────┤
│  POLİTİKA ÖZET        SESSION DURUMU         BÜTÇE          │
│  Allow:  94.5%        Aktif:  2              Token: 12,450  │
│  Deny:   5.3%         Kapanan: 0             Adım:  247/∞   │
│  Human:  0.2%         Uyarı:  1 ⚠️           Maliyet: ~$1.2 │
└─────────────────────────────────────────────────────────────┘
```

#### EKRAN 2 — Karar Gezgini (Decision Explorer)

```
┌─────────────────────────────────────────────────────────────┐
│  KARAR GEZGİNİ                          [Filtrele] [Dışa]  │
├─────────────────────────────────────────────────────────────┤
│  Arama: [place_bet____________] Tarih: [Bugün ▼] Risk: [▼] │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ◉ dec_7f4a...  place_bet  COMPLETED  12:34:01              │
│  ├─ Intent:    EXECUTE_ACTION / FINANCIAL                   │
│  ├─ Risk:      HIGH  ●●●○○                                  │
│  ├─ Kullanıcı: player/usr_4432                              │
│  ├─ Validasyon: ✅ Şema ✅ İş kuralı ✅ Bağlam             │
│  ├─ Politika:  ✅ ALLOW (financial_role_check geçti)        │
│  ├─ Gate:      ✅ PERMIT (idempotency: yeni anahtar)        │
│  ├─ Adapter:   ✅ Supabase RPC place_bet → 201 OK           │
│  └─ Receipt:   a3f8b2c1... [Doğrula]                       │
│                                                             │
│  ◉ dec_2b9c...  transfer_balance  DENIED  12:33:59          │
│  ├─ Intent:    EXECUTE_ACTION / FINANCIAL                   │
│  ├─ Politika:  ❌ UNAUTHORIZED_HIERARCHY                    │
│  ├─ Soft Steer: "Transfer parent→child yönünde olmalı"     │
│  └─ Ajan tepkisi: RE_EVALUATED → başarılı ✅               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### EKRAN 3 — Politika Editörü (Policy Editor)

```
┌─────────────────────────────────────────────────────────────┐
│  POLİTİKA EDİTÖRÜ              Core: v1.2.0  Domain: v0.4  │
├───────────────────────┬─────────────────────────────────────┤
│  KURAL LİSTESİ        │  KURAL DETAYI                       │
│  ─────────────────    │  ─────────────────────────────────  │
│  🔒 Hard Locks (8)    │  Kural: financial_role_check        │
│    └ immutable_state  │  Tür: SOFT (güncellenebilir)        │
│    └ client_direct    │  ─────────────────────────────────  │
│    └ negative_amount  │  Tetikleyici:                       │
│    └ hierarchy_dir    │  category === "FINANCIAL"           │
│    └ ...              │                                     │
│                       │  Koşul:                             │
│  ✏️ Domain Rules (7)  │  !ALLOWED_ROLES.includes(user_role) │
│    └ financial_role ◀ │                                     │
│    └ financial_limit  │  Eylem: DENY                        │
│    └ immutable_coupon │  Kod: UNAUTHORIZED_ROLE             │
│    └ ...              │  Yönlendirme:                       │
│                       │  [_________________________]        │
│  [+ Yeni Kural]       │                                     │
│                       │  Test Et: [Simüle Et]               │
│                       │  Kaydet:  [İmzala ve Uygula]        │
├───────────────────────┴─────────────────────────────────────┤
│  ⚠️ Hard Lock kuralları bu ekrandan değiştirilemez.         │
│     Core güncellemesi için imzalı update paketi gerekir.   │
└─────────────────────────────────────────────────────────────┘
```

#### EKRAN 4 — Session Yöneticisi (Session Manager)

```
┌─────────────────────────────────────────────────────────────┐
│  SESSION YÖNETİCİSİ                   KOBRABET OS / v6.58  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  MEVCUT DURUM                                               │
│  ──────────────────────────────────────────────────────    │
│  Session: #126  │  Versiyon: v6.58  │  Durum: ✅ Aktif     │
│                                                             │
│  AÇIK SORUNLAR                               [+ Ekle]      │
│  🟠 #8   Alt/Üst oranları kilitli — sync atılmadı          │
│  🟠 #9   Beraberlikte iade + çifte şans kilitli            │
│  🟡 #5   Kuponlar muhtemel kazanç font 22→28px             │
│  🟢 #2   profiles_admin_update RLS policy with_check yok   │
│                                                             │
│  SIRADAKİ GÖREVLER                                          │
│  1. G3: OddsButton kilit mantığı   [OddsButton.tsx]        │
│  2. G4: OddsContent veri bağlantısı [OddsContent.tsx]      │
│  3. G5: Market Seed SQL (114 market)                       │
│                                                             │
│  SESSION KAPANIŞ                                            │
│  [session_index.md] [session_log.md] [test_matrix.md]      │
│  [DEPENDENCIES.md]  [Checkpoint Al]  [Onayla ve Kapat]     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### EKRAN 5 — The Bridge (Otomasyon Arayüzü)

```
┌─────────────────────────────────────────────────────────────┐
│  THE BRIDGE — AI ÇIKTISI UYGULAMA MERKEZİ          v0.1   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ADIM 1: AI ÇIKTISINI YAPISTIR                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ { "bundle_id": "session_126_task_1",                 │  │
│  │   "decision": { ... },                               │  │
│  │   "files": [ ... ],                                  │  │
│  │   "migrations": [ ... ] }                            │  │
│  └──────────────────────────────────────────────────────┘  │
│  [Yapıştır]  [Dosyadan Yükle]                              │
│                                                             │
│  ADIM 2: ANA KONTROL (PolicyKernel doğrulaması)            │
│  ✅ JSON Format: Geçerli                                    │
│  ✅ Decision Object: Geçerli                               │
│  ✅ Politika Kontrolü: ALLOW (3 kural geçti)               │
│  ⚠️ Migration: 1 dosya tespit edildi                        │
│                                                             │
│  ADIM 3: FARK GÖRÜNÜMÜ                                     │
│  📄 app/api/place-bet/route.ts  [GÜNCELLEME]               │
│     + 12 satır eklendi, - 3 satır silindi  [Diff Gör]     │
│  🗄️ 20260501120000_add_column.sql  [YENİ]   [Önizle]      │
│                                                             │
│  ADIM 4: UYGULA                                            │
│  [ ] Dosyaları uygula    [Simüle Et]  [Uygula ▶]          │
│  [ ] Migration çalıştır  ⚠️ Önce Supabase'de çalıştır     │
│  [ ] session_log güncelle                                   │
│  [ ] test_matrix güncelle                                   │
│  [ ] Git commit + push   [Hazır olunca]                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 The Bridge Super Bundle Format

AI çıktısını Bridge'e gönderme standardı:

```json
{
  "bundle_id": "session_126_task_1",
  "decision": {
    "intent": "WRITE_DATA",
    "category": "ODDS",
    "payload": { "action_name": "update_odds_button_lock" },
    "context": { "user_role": "admin", "risk_profile": "LOW" },
    "metadata": { "confidence": "HIGH", "self_check_passed": true }
  },
  "files": [
    {
      "operation": "UPDATE",
      "path": "app/odds-overlay/components/OddsButton.tsx",
      "content": "/* tam dosya içeriği — truncated yasak */"
    }
  ],
  "migrations": [],
  "test_updates": [
    {
      "id": "G3-01",
      "scenario": "OddsButton — kilitli selection görünümü",
      "expected": "Kilit ikonu görünür, tıklanamaz",
      "status": "❓"
    }
  ],
  "session_log_block": "## Session 126\n**Görev:** G3 OddsButton kilit mantığı\n**Yapılan:** ...\n**Karar:** ..."
}
```

---

# KISIM IV — UYGULAMA VE TAHMİN

---

## BÖLÜM 8 — YENİ PROJE BOOTSTRAP PROTOKOLÜ

### 8.1 Üç Adımda Başlatma

**Adım 1 — Dosyaları Kopyala**

```bash
# Sovereign Engine OS şablonundan yeni proje başlat
cp -r sovereign-engine-template/ [proje-adi]-os/
cd [proje-adi]-os/

# Proje kimliğini güncelle (CORE.md)
# Stack'i güncelle
# Rol hiyerarşisini güncelle
# Domain Adapter'ı yaz
```

**Adım 2 — AI'ya Boot Komutu Ver**

```
"Benim [PROJE ADI] adında yeni bir projem var.

Sana şu dosyaları veriyorum:
1. SOVEREIGN_ENGINE_OS.md
2. CORE.md ([PROJE ADI] için uyarlanmış)
3. failure_patterns.md (Kobrabet mirası + proje özeli)

Bu andan itibaren:
• Her karar Decision Object formatında üretilecek
• Self-check listesi her çözümden önce çalıştırılacak
• Her session 4 dosya güncellenecek (session_index, session_log, test_matrix, DEPENDENCIES)
• HIGH confidence için self_check_passed=true zorunlu

İlk görev: [PROJE ADI] için FAZ 0 — DB şema tasarımı."
```

**Adım 3 — Domain Adapter'ı Yaz**

```typescript
// domain/[proje]/adapter.ts
class [ProjeAdi]Adapter implements DomainAdapter {
  supports(d: Decision): boolean {
    return d.category === "[PROJE_KATEGORİSİ]";
  }
  async execute(d: Decision): Promise<AdapterResult> {
    // SADECE bu dosyada domain bilgisi var
    // Core katmanlar tamamen domain-agnostik
    switch (d.payload.action_name) {
      // Supabase RPC / REST API çağrıları
    }
  }
}
```

### 8.2 Domain Mapping Şablonu

```
KOBRABET REFERANSI  →  [YENİ PROJE] EŞLEŞTİRMESİ

Finansal İşlemler:
  place_bet         →  [process_payment / book_order / create_reservation]
  settle_coupon     →  [complete_order / release_escrow / confirm_booking]
  transfer_balance  →  [transfer_credits / allocate_budget / move_funds]

Rol Hiyerarşisi:
  admin             →  [owner / super_admin / franchisor]
  superadmin        →  [manager / regional / franchise]
  agent             →  [dealer / branch / operator]
  player            →  [customer / user / client / member]

Veri Katmanları:
  fixtures (statik) →  [products / listings / inventory / schedule]
  odds (orta)       →  [prices / availability / rates / quotes]
  live_odds (RT)    →  [live_inventory / realtime_prices / live_tracking]

Güvenlik Kuralları:
  won/lost kupon    →  [completed_order / processed_payment / confirmed_booking]
  FOR UPDATE kilidi →  [herhangi bir eşzamanlı güncelleme noktası]
  idempotency       →  [herhangi bir finansal veya kritik işlem]
```

### 8.3 Uyarlanmış failure_patterns.md (Yeni Proje)

Her yeni projede asgari korunması gereken 5 evrensel kural:

```markdown
# [PROJE ADI] — FAILURE PATTERNS
> Kobrabet OS'ten miras alınan, proje özelinde uyarlanmış

## 1. DOUBLE EXECUTION (Kobrabet: Double Spend)
Senaryo: Kullanıcı [kritik butona] hızlıca iki kez basar
Önlem: isSubmitting guard (UI) + idempotency_key (DB)

## 2. RACE CONDITION
Senaryo: İki istek aynı anda [kritik kayıt]'a yazıyor
Önlem: FOR UPDATE kilidi, SERIALIZABLE isolation

## 3. PARSİYEL BAŞARISIZLIK
Senaryo: Transaction'ın bir parçası başarısız oldu
Önlem: Tüm işlemler BEGIN/COMMIT/ROLLBACK içinde

## 4. IMMUTABLE STATE
Senaryo: [tamamlanmış kayıt]'a yazma girişimi
Önlem: Policy P4 — locked states listesi

## 5. CLIENT DIRECT
Senaryo: Frontend değeri direkt DB'ye yazıyor
Önlem: Policy P5 — her zaman server-side RPC
```

---

## BÖLÜM 9 — TAHMİN MODELİ (Kobrabet Kalibrasyonu)

### 9.1 Özellik Maliyeti Tablosu

| Özellik Kategorisi | Session Aralığı | Kobrabet Gerçekleşme |
|--------------------|-----------------|----------------------|
| Auth sistemi (giriş + rol) | 8–15 | Session 1–12 |
| Basit CRUD | 1–2 | — |
| Atomik finansal RPC | 3–5 | place_bet |
| Hiyerarşik yetki (4 seviye) | 5–10 | admin/super/agent/player |
| Realtime özellik | 4–8 | live_odds, kupon bildirimi |
| Harici API entegrasyonu | 5–10 | The Odds API |
| Karmaşık UI (state + tabs) | 3–6 | odds-overlay 6 tab |
| Migration + seed | 1–2 | market_definitions (114 market) |
| Kritik bug fix | 1–3 | odds sync timeout |
| Minor bug fix | 0.5–1 | — |
| Admin paneli (tam) | 12–18 | yönetim dashboard |
| Realtime canlı sistem | 8–15 | live bahis altyapısı |

### 9.2 Proje Büyüklük Kategorileri

```
MİKRO PROJE (< 20 session)
  • 3-5 CRUD endpoint
  • 1 rol seviyesi
  • Harici API yok
  • Realtime yok
  Tavsiye: Sovereign Engine overkill — standart boilerplate yeterli

KÜÇÜK PROJE (20–50 session)
  • 10-20 endpoint
  • 2-3 rol seviyesi
  • 1 harici API
  • Opsiyonel realtime
  Tavsiye: Sovereign Engine ideal başlangıç noktası ✅

ORTA PROJE (50–100 session)
  • 20-50 endpoint
  • 4 rol seviyesi
  • 2-3 harici API
  • Realtime zorunlu
  Tavsiye: Sovereign Engine tam kapasitede ✅✅

BÜYÜK PROJE (100–200 session)
  • 50+ endpoint
  • Karmaşık hiyerarşi
  • Birden fazla harici API
  • Realtime + WebSocket
  Tavsiye: Sovereign Engine + Rust core zorunlu ✅✅✅

ENTERPRISE (200+ session)
  • Çok-tenant mimari
  • Regulatory compliance
  • ML/AI pipeline
  Tavsiye: Sovereign Engine + mikroservis adaptasyon gerekli
```

### 9.3 Tahmin Formülü

```
TOPLAM_SESSION = (FAZ_0 + FAZ_1 + FAZ_2) × GECIKME_KATSAYISI

FAZ_0 (Altyapı):
  Basit:    10–20 session
  Orta:     20–35 session
  Karmaşık: 30–50 session

FAZ_1 (Core Domain):
  Basit:    15–25 session
  Orta:     25–45 session
  Karmaşık: 40–65 session

FAZ_2 (Gelişmiş):
  Basit:    10–20 session
  Orta:     20–35 session
  Karmaşık: 30–50 session

GECIKME_KATSAYISI:
  Yeni domain + yeni stack:         1.45
  Kısmen bilinen domain:            1.30
  Kanıtlanmış domain (Kobrabet):    1.18
  Sovereign Engine ile (bu sistem): 1.12  ← %6 iyileşme

KOBRABET DOĞRULAMASI:
  Beklenen:     (20 + 45 + 30) × 1.30 = ~123 session
  Gerçekleşen:  125 session ✓
```

### 9.4 Hızlı Tahmin Aracı

```
Aşağıdaki kutuları işaretle ve topla:

[ ] Kaç finansal RPC?           × 4 session each
[ ] Kaç CRUD endpoint?          × 1.5 session each
[ ] Realtime gerekiyor?         +20 session
[ ] Harici API entegrasyonu?    × 7 session each
[ ] Hiyerarşi kaç seviye?       × 5 session each
[ ] Yönetim paneli?             +15 session
[ ] Mobil-öncelikli UI?         × 1.2 multiplier

Toplam × Gecikme katsayısı = Tahmini session

Örnek (KOBRABET):
  5 finansal RPC: 20
  30 CRUD:        45
  Realtime:       20
  1 API:           7
  4 rol seviyesi: 20
  Admin panel:    15
  Mobile UI:      × 1.2
  Alt toplam:     127 × 1.2 = 152... ama Sovereign Engine katsayısı 1.12 → 127 × 1.12 = ~142
  Gerçek: 125 (Sovereign Engine overhead azalttı ✓)
```

---

## BÖLÜM 10 — SİSTEM SAĞLIK PANELİ (Her Session Başında)

### 10.1 Otomatik Sağlık Kontrolü

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SOVEREIGN ENGINE SAĞLIK KONTROLÜ — #[SESSION_NO]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[ ] session_index.md'de 🔴 kritik görev var mı?
    → EVET: önce bu görev

[ ] Son migration deploy edildi mi?
    → HAYIR: Production ile dev DB'si ayrışıyor ⚠️

[ ] test_matrix'te ❓ kritik test var mı?
    → EVET: deploy öncesi test edilmeli

[ ] Bekleyen ASK_HUMAN kararı var mı?
    → EVET: Dashboard'dan manuel onay gerekiyor

[ ] ARCHITECTURE.md son DB değişikliğini yansıtıyor mu?
    → HAYIR: Güncellenmesi gerekiyor

[ ] failure_patterns.md'ye eklenmemiş yeni hata var mı?
    → EVET: Bu session'da ekle

ÇIKTI FORMAT:
"Sağlık: ⚠️ [N] açık — [kısa liste]. Devam edelim mi?"
"Sağlık: ✅ Temiz. Sıradaki: [görev]"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## BÖLÜM 11 — KRİTİK ÖĞRENME ÇIKARTMALARI

### 11.1 Kesinlikle Yapılacaklar

```
✅ Finansal işlemlerin TAMAMI → server-side RPC (Supabase function)
✅ Her RPC → tek transaction (BEGIN / COMMIT / ROLLBACK)
✅ Eş zamanlı yazma riski → FOR UPDATE kilidi
✅ Mükerrer istek riski → idempotency_key
✅ Platform limitini FAZ 0'da belirle (Vercel Hobby = 10s timeout!)
✅ Env variable listesini proje başında tamamla
✅ Migration: önce dosya → sonra çalıştır (asla tersine)
✅ ARCHITECTURE.md → her DB değişikliğinde güncelle
✅ test_matrix → her özellik için senaryo eklenmeden deploy yapılmaz
✅ session_log → her görevde yaz, "sonra yazarım" olmaz
✅ failure_patterns → her yeni hata kalıbı belgelenir
✅ DEPENDENCIES.md → her yeni dosya/bağımlılık eklenir
```

### 11.2 Kesinlikle Yapılmayacaklar

```
❌ Client değerini direkt DB'ye yazma
❌ won/lost/settled/cancelled kayıtlara yazma
❌ Türkçe karakter içeren route (URL encoding sorunu)
❌ Tüm sistemi tek session'da yükleme
❌ Eksik veriyle HIGH confidence verme
❌ Onaysız kod değişikliği
❌ Truncated çıktı (her zaman tam dosya)
❌ Birden fazla sorunu aynı anda çözme
❌ Validasyon olmadan production deploy
❌ failure_patterns.md okumadan finansal işlem yazma
```

### 11.3 Beklenmedik Gecikme Kaynakları

| Risk | Kobrabet Kanıtı | Erken Tespit | Maliyet |
|------|-----------------|--------------|---------|
| Platform timeout | Vercel Hobby 10s | FAZ 0'da tek endpoint test et | +3 session |
| Eksik env variable | odds sync patladı | .env.example'ı tam yaz | +2–5 session |
| Bayat dokümantasyon | ARCHITECTURE drift | Her session kapanışında güncelle | +1–3 session |
| Migration çakışması | — | Numaralı + sıralı migrationlar | +2 session |
| Race condition (keşfedilmemiş) | place_bet double spend | failure_patterns.md FAZ 1'de oku | +3–8 session |
| Türkçe karakter route | market isimleri | ASCII-only zorunlu kural | +2 session |

---

## EKLER

### A. Teknik Yığın (Sovereign Engine v2.0)

| Katman | Teknoloji | Güncelleme |
|--------|-----------|-----------|
| **Rust Core** | Rust + Tokio + napi-rs + sha2 + ed25519 | Sahip imzalı update |
| Frontend | Next.js App Router + Tailwind CSS | Kullanıcı günceller |
| Dashboard | Tauri + React + Zustand | Kullanıcı günceller |
| Şema Doğrulama | Zod (TS) / Pydantic (Python) | Kullanıcı günceller |
| Backend/Auth | Supabase (PostgreSQL + RLS + Realtime) | Kullanıcı günceller |
| İletişim (Dahili) | gRPC/Protobuf | Sabit |
| İletişim (Harici) | JSON-RPC 2.0 / MCP | Kullanıcı günceller |
| Cache/Idempotency | In-memory → Redis (ölçekte) | Kullanıcı günceller |
| Kalıcı Log | PostgreSQL (hash chain) | Sabit |
| Kimlik | OAuth 2.1 / SPIFFE | Kullanıcı günceller |
| İzleme | OpenTelemetry + Grafana | Kullanıcı günceller |
| Sandboxing | Docker → gVisor (prod) | Proje büyüklüğüne göre |
| Deploy | Vercel (Hobby→Pro sınırına dikkat!) | Kullanıcı günceller |

### B. Rust Core Güncellenebilirlik Mimarisi

```
SAHİBİN GÜNCELLEME AKIŞI:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Sahip yeni kural/hata düzeltmesi yazar (Rust)
   ↓
2. Derlenir ve ed25519 private key ile imzalanır
   (Private key sahipte, asla paylaşılmaz)
   ↓
3. update.manifest dosyası güncellenir
   ↓
4. Kullanıcı dashboard'dan güncelleme alır:
   "Yeni güncelleme mevcut v1.2.1 — Uygula?"
   ↓
5. Binary indirilir, imza doğrulanır (public key hard-coded)
   ↓
6. Donanım parmak izi kontrolü
   ↓
7. Yeni binary aktif edilir

KOPYALANAMAZ:
━━━━━━━━━━━━━
• Binary başka donanımda çalışmaz (parmak izi)
• Kaynak kodu paylaşılmaz
• Reverse engineering: Rust + optimizasyon flag'leri + obfuscation

GÜNCELLENEBİLİR:
━━━━━━━━━━━━━━━━
• Sahip imzalı update paketi gönderir
• Soft rules güncellenebilir
• Hard locks hiçbir zaman değiştirilemez
• Kullanıcı kendi domain adapter'ını her zaman günceller
```

### C. Versiyon Şeması

```
SOVEREIGN ENGINE vX.Y.Z
  X → Büyük mimari değişiklik (Rust core versiyonu)
  Y → Yeni özellik (domain adapter, UI panel)
  Z → Bug fix / kural güncellemesi

PROJE VERSİYONU vFAZ.MİLESTONE
  Kobrabet formatı: v6.58 = Faz 6, 58. milestone
  Tavsiye: Her projede bu format korunur
```

### D. Terim Sözlüğü

| Bu Sistemdeki Terim | Meta-Framework Karşılığı | Kobrabet Karşılığı |
|---------------------|--------------------------|--------------------|
| PolicyKernel | Policy Engine (OPA/Rego) | failure_patterns.md kuralları |
| AuditChain | Hash Chain Log | session_log.md |
| ConstitutionGuard | AI Runtime Constitution | CORE.md + AI_AGENT.md |
| PreFlightGate | Pre-flight Read | place_bet öncesi bakiye kontrolü |
| SideEffectTracer | Side-effect Map | DEPENDENCIES.md |
| Soft Steer | Reject + redirect mesajı | "yetkisiz — şunu dene" yanıtı |
| Decision Trace | 5 katmanlı denetim yapısı | session_log entry |
| CAR | Canonical Action Representation | Decision Object formatı |
| Hard Lock | Değiştirilemez kural | won/lost kupona dokunma yasağı |
| The Bridge | Execution automation | Manuel kopyala-yapıştır yerine |

---

*SOVEREIGN ENGINE OS v2.0*  
*Kobrabet OS v6.58 (125 session) + Meta-Framework v1.0 + Rust Core Tasarım Planı*  
*Oluşturulma: 2026-04-30 | Kalibrasyon: KOBRABET üretim verisi*  
*Çekirdek dil: Rust | Arayüz: Tauri/React | Adaptör: TypeScript*
