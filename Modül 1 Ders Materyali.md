---
# OpenCode Agents - Modül 1: Ders Materyali

> **Konu:** Builtins, Permissions Mekanizması ve Override Mantığı  
> **Ders Tarihi:** Ocak 2026  
> **Doküman Sürümü:** v1.0  
> **Kaynak:** OpenCode V2 Resmi Dokümantasyonu [1]

---

## 📑 İçindekiler

1. Agent Nedir?
2. Builtins (Hazır Agent'lar)
3. Permissions Sistemi: Üçlü Yapı (action, resource, effect)
4. Wildcard (*) Kullanımı
5. Override Mekanizması: "Son Eşleşen Kazanır"
6. Sıralama Stratejileri
7. Özet ve Kontrol Listesi

---

## 1. Agent Nedir?

Bir agent; **sistem prompt + model tercihi + izinler + görsel detayları** bir araya getirerek oluşturulan, isimlendirilmiş asistan profilidir [1].

### Temel Bileşenler

| Bileşen | Açıklama |
|---------|----------|
| Sistem prompt | Agent'ın nasıl davranacağını belirleyen talimatlar |
| Model tercihi | Hangi LLM'in kullanılacağı (örn. `anthropic/claude-sonnet-4-5#high`) |
| Permissions | Hangi tool'lara, hangi komutlara/dosyalara izin verildiği |
| Görsel detaylar | Renk, açıklama, görünürlük |

---

## 2. Builtins (Hazır Agent'lar)

OpenCode dört tane "görünür" hazır agent ile birlikte gelir [1]. Bunlar binary içinde tanımlıdır ve doğrudan erişilemez, ancak **aynı ID ile override edilebilir**.

### 2.1 Build (`build`)

| Özellik | Değer |
|---------|-------|
| **Mod** | primary |
| **Amaç** | Varsayılan kodlama agent'ı |
| **Araçlar** | Varsayılan olarak izinlidir [1] |
| **Hassas dosyalar** | `.env` vb. okuma → **onay ister** [1] |
| **Workspace dışı erişim** | **Onay ister** [1] |

### 2.2 Plan (`plan`)

| Özellik | Değer |
|---------|-------|
| **Mod** | primary |
| **Amaç** | Keşif ve planlama |
| **Dosya düzenleme** | Normal proje dosyalarını düzenlemez [1] |
| **Plan dosyaları** | İstendiğinde OpenCode plan dosyalarını yazabilir [1] |
| **Shell komutları** | Permission kontrolündedir [1] |

### 2.3 General (`general`)

| Özellik | Değer |
|---------|-------|
| **Mod** | subagent |
| **Amaç** | Araştırma ve çok adımlı iş |
| **Tool erişimi** | Geniş [1] |
| **Subagent başlatma** | Yapamaz [1] |
| **Çalışma** | Taze context ile, ana oturumdan bağımsız [1] |

### 2.4 Explore (`explore`)

| Özellik | Değer |
|---------|-------|
| **Mod** | subagent |
| **Amaç** | Kod veya web kaynaklarını arama/okuma |
| **Dosya düzenleme** | Yapmaz [1] |
| **Subagent başlatma** | Yapamaz |

### 2.5 Gizli Agent'lar

Hidden compaction, title ve summary agent'ları arka planda bakım işleri yapar; bunları doğrudan seçemezsiniz [1]. V2'de scout agent'ı yoktur [1].

### 2.6 Builtin İlişki Diyagramı

```
Kullanıcı
   ↓
[build] veya [plan]   ← primary (doğrudan kullanılır)
   ↓ subagent tool'u ile
   ├── [general]       ← çok adımlı araştırma
   └── [explore]       ← arama/okuma
```

---

## 3. Permissions Sistemi: Üçlü Yapı

Her permission kuralı üç temel alandan oluşur [1]:

```jsonc
{
  "action":   "shell",         // Hangi tool ailesi?
  "resource": "git push *",    // Hangi komut/yol/agent ID?
  "effect":   "ask"            // Ne yapılsın?
}
```

### 3.1 Action Değerleri

Action, hangi tool çağrısının kontrol edileceğini belirler [1].

| Action | Kapsam | Ne Zaman Tetiklenir |
|--------|--------|---------------------|
| `shell` | Tüm kabuk komutları [1] | `npm install`, `git push` çalıştırılırken |
| `edit` | `edit`, `write`, `patch` tool'ları [1] | Dosya oluşturma/değiştirme/silme |
| `subagent` | Çocuk agent başlatma [1] | Ana agent `subagent` tool'unu çağırdığında |
| `read` | `read` tool'u [1] | Dosya içeriği okuma |
| `glob` | `glob` tool'u [1] | Pattern ile dosya arama |
| `grep` | `grep` tool'u [1] | İçerik arama |
| `webfetch` | `webfetch` tool'u [1] | URL'den içerik çekme |
| `websearch` | `websearch` tool'u [1] | Web araması |
| `skill` | Skill yükleme [1] | Skill dosyası yüklenirken |
| `external_directory` | Workspace dışı erişim | Proje klasörünün dışına çıkarken |
| `*` | HEPSI | Yukarıdakilerin tümü |

### 3.2 Resource Tanımlama Kalıpları

Resource, action'a göre değişen kalıplar kullanır [1]:

#### Shell Action → Komut Metni (Ham)

Shell için `~` ve `$HOME` genişletmesi **yapılmaz** [1]:

```jsonc
{ "action": "shell", "resource": "git status",    "effect": "allow" }
{ "action": "shell", "resource": "git push *",    "effect": "ask"   }
{ "action": "shell", "resource": "* npm *",       "effect": "ask"   }
{ "action": "shell", "resource": "*",             "effect": "deny"  }
```

#### Dosya Action'ları (`read`, `edit`, `external_directory`) → Glob Pattern

Bu action'lar için `~` ve `$HOME` **genişletilir** [1]:

```jsonc
{ "action": "read",  "resource": "/etc/passwd",   "effect": "deny"  }
{ "action": "read",  "resource": "~/notes/**",    "effect": "allow" }
{ "action": "edit",  "resource": "src/**/*.ts",   "effect": "allow" }
{ "action": "edit",  "resource": "package.json",  "effect": "ask"   }
```

#### Subagent Action → Agent ID

```jsonc
{ "action": "subagent", "resource": "reviewer",  "effect": "allow" }
{ "action": "subagent", "resource": "*",         "effect": "deny"  }
{ "action": "subagent", "resource": "team/*",    "effect": "allow" }
```

### 3.3 Effect Değerleri

Effect üç değer alabilir [1]:

| Effect | Davranış | Kullanım Alanı |
|--------|----------|----------------|
| `allow` | Tool çağrısı otomatik onaylanır, kullanıcıya sorulmaz [1] | Güvenli ve sık kullanılan komutlar |
| `ask` | Her çağrıda kullanıcıya onay sorusu gösterilir [1] | Hassas ama bazen gerekli komutlar |
| `deny` | Tool çağrısı reddedilir, model alternatif yönteme yönlendirilir [1] | Tehlikeli veya istenmeyen komutlar |

---

## 4. Wildcard (*) Kullanımı

Hem action hem resource wildcard destekler [1].

### 4.1 Action'da Wildcard

```jsonc
// Tüm action'ları etkiler
{ "action": "*", "resource": "*", "effect": "deny" }

// Sadece belirli bir action ailesini
{ "action": "shell", "resource": "*", "effect": "ask" }
```

### 4.2 Resource'ta Wildcard — Pozisyonlar

| Kalıp | Anlam | Örnek Eşleşme |
|-------|-------|---------------|
| `"*"` | Her şey | Tüm komutlar/dosyalar |
| `"git *"` | `git` ile başlayan | `git status`, `git push origin` |
| `"*.ts"` | `.ts` ile biten | `app.ts`, `index.ts` |
| `"src/**"` | `src` altındaki her şey | `src/app.js`, `src/utils/index.ts` |
| `"**/*.test.*"` | Herhangi bir yerde `.test.` içeren | `src/app.test.ts` |
| `"npm *"` | `npm` ile başlayan | `npm install`, `npm test` |
| `"* push *"` | İçinde `push` geçen | `git push origin`, `docker push img` |

### 4.3 HOME Genişletmesi: Kritik Fark

| Action Tipi | `~` ve `$HOME` Davranışı |
|-------------|--------------------------|
| `read`, `edit`, `external_directory` | **Genişletilir** [1] |
| `shell` | **Genişletilmez** (ham metin olarak kalır) [1] |

Bu yüzden:

```jsonc
// ✅ Bu çalışır — read HOME'ı anlar
{ "action": "read", "resource": "~/notes/**", "effect": "allow" }

// ❌ Bu beklendiği gibi çalışmaz — shell için genişletme yok
{ "action": "shell", "resource": "cat ~/notes/*", "effect": "allow" }
```

---

## 5. Override Mekanizması: "Son Eşleşen Kazanır"

### 5.1 Temel Kurallar

OpenCode'un permission sistemi üç temel ilkeye dayanır [1]:

1. **"The last matching rule wins"** — Son eşleşen kural kazanır [1]
2. **"Permission rules append"** — Override'lar sona eklenir [1]
3. **"Global permissions apply before agent-specific rules"** — Genel izinler agent kurallarından önce uygulanır [1]

### 5.2 Override Nasıl Çalışır?

Bir agent'ı override ettiğinizde:

- Orijinal kurallar **silinmez**
- Yeni kurallar listenin **sonuna eklenir** [1]
- OpenCode listeyi yukarıdan aşağı tarar ve **en son eşleşen** kuralı uygular

### 5.3 Görsel Örnek: Override Senaryosu

OpenCode'un dahili build permissions listesi (varsayımsal):

```
1. shell, "git *",       allow   ← tüm git komutlarına izinli
2. shell, "*",           ask     ← diğer shell komutları onay ister
3. read, "*",            allow   ← tüm okumalara izinli
4. edit, "*",            allow   ← tüm düzenlemelere izinli
5. external_directory,*, ask     ← workspace dışı onay ister
```

Sizin override'ınız:

```jsonc
{
  "agents": {
    "build": {
      "permissions": [
        { "action": "shell", "resource": "git status", "effect": "deny" }
      ]
    }
  }
}
```

**Birleşik (Merged) Liste:**

```
1. shell, "git *",       allow   ← OpenCode dahili
2. shell, "*",           ask     ← OpenCode dahili
3. read, "*",            allow   ← OpenCode dahili
4. edit, "*",            allow   ← OpenCode dahili
5. external_directory,*, ask     ← OpenCode dahili
6. shell, "git status",  deny    ← SİZİN override'ınız (sona eklendi)
```

`git status` çağrıldığında OpenCode yukarıdan aşağı tarar:

| Adım | Kural | Eşleşiyor mu? | Uygulanan mı? |
|------|-------|---------------|---------------|
| 1 | `shell, "git *", allow` | Evet | Değil (sonra bakılacak) |
| 2 | `shell, "*", ask` | Evet | Değil (sonra bakılacak) |
| 3-5 | shell değil | Hayır | — |
| 6 | `shell, "git status", deny` | Evet | **BU UYGULANIR** |

**Sonuç:** `git status` → ❌ **deny**

### 5.4 Neden Sondaki Kazanır?

Bu strateji şunu sağlar:

- **Override'lar her zaman sona eklenir** → daha spesifik yazarsanız kazanır
- **Genel kurallar önce gelir** → sonradan eklenen kurallarla daraltılabilir
- **Tahmin edilebilirlik** → sıralama kontrolü sizde

---

## 6. Sıralama Stratejileri

### 6.1 Doğru Strateji: "Geniş Önce, İstisna Sonra"

```jsonc
{
  "permissions": [
    { "action": "*", "resource": "*", "effect": "deny" },         // 1. Geniş kural önce
    { "action": "read", "resource": "src/**", "effect": "allow" } // 2. Spesifik istisna sonra
  ]
}
```

Sonuç:

| Tool Çağrısı | Ne Olur? | Neden? |
|--------------|----------|--------|
| `src/app.js` oku | ✅ allow | Kural 2 eşleşir (son eşleşen) |
| `package.json` oku | ❌ deny | Kural 1 eşleşir (son eşleşen) |
| `src/app.js` düzenle | ❌ deny | Kural 1 eşleşir (son eşleşen) |

### 6.2 Yanlış Strateji: "İstisna Önce, Geniş Sonra"

```jsonc
// ❌ YANLIŞ — istisna geniş kural tarafından ezilir
{
  "permissions": [
    { "action": "read", "resource": "src/**", "effect": "allow" }, // 1. İstisna önce
    { "action": "*", "resource": "*", "effect": "deny" }           // 2. Geniş kural SONDA
  ]
}
```

Bu yapıda `src/**` için `allow` asla uygulanmaz; çünkü son eşleşen `* * deny` her zaman kazanır.

### 6.3 Altın Kural

> **Geniş kurallar listenin başında, dar istisnalar listenin sonunda.**  
> Çünkü son eşleşen kural kazanır [1].

---

## 7. Pratik Örnekler

### 7.1 Read-Only Agent

```markdown
---
description: Sadece okur, asla düzenlemez
mode: subagent
permissions:
  - action: "*"
    resource: "*"
    effect: deny
  - action: read
    resource: "src/**"
    effect: allow
  - action: glob
    resource: "src/**"
    effect: allow
---
```

### 7.2 Güvenli Build Profili

```jsonc
{
  "agents": {
    "build": {
      "permissions": [
        { "action": "shell", "resource": "*",            "effect": "ask"    },
        { "action": "shell", "resource": "ls *",         "effect": "allow"  },
        { "action": "shell", "resource": "cat *",        "effect": "allow"  },
        { "action": "shell", "resource": "git status",   "effect": "allow"  },
        { "action": "shell", "resource": "git log *",    "effect": "allow"  },
        { "action": "shell", "resource": "rm -rf *",     "effect": "deny"   },
        { "action": "shell", "resource": "sudo *",       "effect": "deny"   },
        { "action": "read",  "resource": ".env",         "effect": "deny"   },
        { "action": "read",  "resource": "~/.ssh/**",    "effect": "deny"   }
      ]
    }
  }
}
```

### 7.3 Subagent Orkestasyonu

```jsonc
{
  "agents": {
    "orchestrator": {
      "permissions": [
        { "action": "subagent", "resource": "*",         "effect": "deny"  },
        { "action": "subagent", "resource": "reviewer",  "effect": "allow" }
      ]
    }
  }
}
```

**Mantık:** "Tüm subagent'ları yasakla, sadece `reviewer`'a izin ver."

---

## 8. Özet ve Kontrol Listesi

### Temel Kavramlar
- [x] Agent; sistem prompt + model + izinler + görsel detaylar birleşimidir [1]
- [x] Builtin'lar binary içinde tanımlı, `.opencode/`'da görünmez ama override edilebilir [1]

### Builtins
- [x] **build**: primary, varsayılan kodlama, tool'lar izinli [1]
- [x] **plan**: primary, keşif ve planlama, normal dosyaları düzenlemez [1]
- [x] **general**: subagent, araştırma ve çok adımlı iş, subagent başlatamaz [1]
- [x] **explore**: subagent, arama/okuma, dosya düzenlemez [1]
- [x] V2'de scout yok; hidden agent'lar (compaction, title, summary) doğrudan seçilemez [1]

### Permissions Üçlüsü
- [x] Action türleri: `shell`, `edit`, `read`, `glob`, `grep`, `webfetch`, `websearch`, `subagent`, `skill`, `external_directory`, `*` [1]
- [x] Resource; action'a göre komut metni, dosya yolu veya agent ID olur [1]
- [x] Effect üç değer: `allow`, `ask`, `deny` [1]

### Wildcard ve Sıralama
- [x] Wildcard `*` hem action'da hem resource'ta kullanılır [1]
- [x] Resource'ta başta/ortada/sonda wildcard olabilir [1]
- [x] `~` ve `$HOME` shell için genişletilmez, dosya action'ları için genişletilir [1]
- [x] **Son eşleşen kural kazanır** [1]
- [x] **Permission kuralları sona eklenir (append)** [1]
- [x] **Global izinler agent kurallarından önce uygulanır** [1]
- [x] **Geniş kurallar önce, istisnalar sonra** yazılmalı [1]

---

## 9. Referanslar

[1] OpenCode V2 Resmi Dokümantasyonu - Agents  
https://opencode.ai/v2/docs/agents/

---

*Bu materyal, OpenCode V2 dokümantasyonundaki "Agents" başlığındaki bilgilere dayanmaktadır [1]. Hands-on deneyler ve soru-cevaplar ile desteklenmiştir.*