# miracle-skills

Tıklatbari / BeyLink projelerinden çıkarılmış, yeniden kullanılabilir mühendislik **skill**'leri.
Bu repo **hem bir Claude Code plugin'i hem de kendi marketplace'i**dir — tek yerden hem yerelde
(terminal) hem de claude.ai / Claude Desktop'ta kurulur.

## İçerdiği skill'ler

| Skill | Ne işe yarar |
|---|---|
| **link-redirect-tracking** | URL kısaltıcı yönlendirme motoru + tıklama takibi + BTAG (affiliate) atıfı. Sıralı redirect pipeline, async loglama (geoip+UA), denormalize sayaç ↔ analytics ilişkisi, temiz-URL BTAG two-hop döngüsü. |
| **analytics-panel** | Plan-gated, timezone-uyumlu tıklama/olay analitik paneli (SQLite). Tarih aralığı filtreleri, plan bazlı özellik kilitleme, geo/cihaz kırılımları, CSV export, paylaşılabilir public rapor. |
| **panel-timezone-date-ranges** | Kullanıcı seçmeli panel saat dilimi (Yerel/İstanbul saat + toggle) → timezone-tutarlı tarih aralığı filtreleri ve CSV export. Tek UTC offset tüm tarih sorgularına akar. |
| **usdt-trc20-payments** | Sağlayıcısız, self-custody USDT (TRC-20) kripto ödeme + kredi yükleme. TronGrid REST doğrulama, ekstra npm bağımlılığı yok, private key sunucuda değil. |

## Kurulum

### 1. Marketplace'i ekle (bir kez)
```
/plugin marketplace add tiklabari-cpu/miracle-skills
```

### 2. Plugin'i kur
```
/plugin install miracle-skills@miracle-skills
```

Kurduktan sonra dört skill de her projede otomatik olarak kullanılabilir hale gelir
(Claude ilgili işi görünce kendisi devreye alır).

### Güncelleme
Repoya yeni commit push ettikten sonra:
```
/plugin marketplace update miracle-skills
```

## Yapı
```
miracle-skills/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifesti
│   └── marketplace.json    # marketplace manifesti (repo kendini yayınlar)
├── skills/
│   ├── analytics-panel/SKILL.md
│   ├── link-redirect-tracking/SKILL.md
│   ├── panel-timezone-date-ranges/SKILL.md
│   └── usdt-trc20-payments/SKILL.md
└── README.md
```

## Lisans
MIT
