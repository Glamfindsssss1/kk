# StyleFinder — Projekt-Briefing für eine neue Claude-Sitzung

Dies ist eine vollständige Zusammenfassung des Projekts, wie es in einer vorherigen, langen Arbeitssitzung erarbeitet wurde. Ziel: eine neue Claude-Sitzung soll ohne erneutes Erkunden sofort produktiv weiterarbeiten können.

## Was ist StyleFinder?

Eine Single-File Web-App (`index.html`, ~10.000 Zeilen HTML/CSS/JS, kein Build-System) — eine Creator-Linkpage für Fashion-/Resell-Creators, ähnlich Linktree, aber spezialisiert auf Kleidung aus China (Weidian-Marktplatz). Jeder Creator bekommt eine eigene Seite über `?creator=slug`.

**Geschäftsmodell:** Creator bewerben Produkte, Besucher klicken über Affiliate-/Promo-Links durch zu Registrierungs-/Shop-Links (z.B. lovegobuy.com, hipobuy.com, usfans.com — Weidian-Weiterleitungs-/Fulfillment-Dienste). Kleinere Creator teilen sich oft zu dritt (max. 3) einen echten Weidian-Shop, weil sich Einzel-Tracking für sie nicht lohnt. Zielgröße laut Betreiber: ~100 Creator, jeweils bis zu 4000 Produkte, wenige echte Admin-Zugänge (nicht 100).

## Repo & Hosting

- GitHub: `Glamfindsssss1/kk`, Branch `claude/weitermachen-20ezck`
- Aktuell lokal ausgecheckt unter `/home/user/kk`
- Hosting: GitHub Pages (ein anderes verwandtes Repo `Warriorfindss2` wurde früher erwähnt — aktueller Live-Stand unklar, im Zweifel nachfragen)
- Backend: Supabase-Projekt `voskcgnbwpemacphyuxb` (eu-west-1), Postgres 17

## Datenbank-Schema (public-Schema)

| Tabelle | Zweck |
|---|---|
| `creators` | Profil pro Creator: Name, Slug, Avatar, Theme-Farbe, Social-Links (TikTok/Instagram), Affiliate-Button-Konfig, How-to-buy-Link, `weidian_shop_id` |
| `products` | Produkte pro Creator: Titel, Preis, Bild, Kategorie/Style/Subcat, Rabatt-Badge, `group_id` (für geteilte Kataloge über mehrere Creator) |
| `categories` / `styles` / `subcategories` | Pro Creator individuell ODER global (`creator_id IS NULL` = Standard-Vorlage, wird per `apply_default_categories_to_creator()` kopiert) |
| `site_settings` | Globaler Standard für den "How to buy"-Button |
| `analytics_events` | Seitenaufrufe/Banner-Klicks/Kauf-Klicks, anonym einfügbar (RLS `anyone can insert`) |
| `weidian_credentials` | Weidian App Key/Secret pro Shop, nur Super-Admin, `last_synced_at` für inkrementellen Sync |
| `weidian_orders` | Echte synchronisierte Bestellungen (Umsatz-Dashboard) |
| `affiliate_agents` | Konfigurierbare Affiliate-Partner (Host-Muster, URL-Template, Invite-Code-Parameter) |
| `admins` / `admin_creators` | Admin-Rollen: `is_super_admin` sieht alles, Unter-Admins nur ihre zugewiesenen Creator (per RLS durchgesetzt, nicht nur UI) |
| `live_sessions` | Kurzlebige "gerade online"-Heartbeats (75s-Fenster) |

RLS ist auf **jeder** Tabelle aktiv. Storefront liest Creator-Daten nie direkt aus `creators`, sondern über die RPC `get_creator_by_slug()`/`get_default_creator()` (SECURITY DEFINER, umgeht bewusst die gesperrte Public-Read-Policy).

## Admin-Zugang

Versteckt: Suchfeld öffnen → "george clooney" eingeben → Währung EUR + Sprache DE → Login-Formular erscheint. Login läuft über Benutzername statt E-Mail (intern in eine feste Fake-Mail-Domain `@stylefinder-admin.internal` umgewandelt, `sanitizeAdminUsername()` im Frontend UND in der Edge Function `manage-admins` — **beide müssen synchron bleiben**, sonst passt der Login nicht mehr zusammen).

## Edge Functions (Supabase)

- **`weidian-sync`** — holt Bestellungen von Weidian. Läuft per pg_cron alle 6 Minuten für alle aktiven Shops (Batches à 5 gleichzeitig). Seit dieser Session **inkrementell**: merkt sich `last_synced_at` pro Shop, fragt nur noch Neues ab (+ 30 Min Überlappungspuffer) statt bei jedem Takt die vollen letzten 2 Monate neu zu holen. `MAX_PAGES=40` bleibt als Sicherheitsnetz, markiert aber `truncated:true` im Ergebnis statt still Daten zu verlieren.
- **`manage-admins`** — Chef-Admin verwaltet Unter-Admins (Anlegen/Löschen/Passwort zurücksetzen/Creator-Zuweisung), nutzt Service-Role-Key nach eigener `is_super_admin()`-Prüfung.
- **`weidian-oauth-start`/`weidian-oauth-callback`** — stillgelegt (HTTP 410), technisch nicht löschbar über die verfügbaren Tools, harmlos.

## In dieser Session behobene Bugs

1. **`index.html` versehentlich geleert** (Commit `a3df28f` hatte den gesamten Inhalt gelöscht) — aus einer vom Nutzer hochgeladenen Datei wiederhergestellt.
2. **Wechselkurse waren fest/veraltet** (EUR/USD 1.09, EUR/CNY 7.85) — jetzt Live-Abruf von frankfurter.app, 24h gecacht, Fallback auf alte Werte bei Netzwerkfehler.
3. **Umsatz-Dashboard zeigte Fake-Demo-Zahlen**, sobald der gewählte Zeitraum (z.B. "Heute" direkt nach Mitternacht Peking-Zeit) zufällig 0 echte Bestellungen hatte — `hasRealData` prüfte nur den aktuellen Zeitraum statt "gab es JEMALS echte Daten". Gefixt: eigene Existenzprüfung unabhängig vom Zeitraum-Filter.
4. **Weidian-Sync war nicht inkrementell** (siehe oben) — behoben.
5. **Analytics "Gesamt"-Ansicht hatte kein Zeilenlimit** — gleiches Muster wie Bug 3, aber für `analytics_events`: bei "Gesamt" + "Alle Creator" gab es keine `.gte()`-Filterung UND kein Limit → Risiko des stillen Abschneidens durch Supabases Standard-Zeilenbegrenzung bei wachsendem Datenvolumen. Gefixt: sortiert absteigend + `.limit(20000)` als Sicherheitsnetz.
6. **Analytics-Events ließen sich unbegrenzt fluten** (Fake-Traffic über beliebig viele zufällige Geräte-IDs) — einfache DB-Trigger-Bremse ergänzt: max. 40 Events/Minute pro Geräte-ID, stillschweigend verworfen darüber (bewusst nur ein Schutz gegen naives Fluten, kein Schutz gegen einen Angreifer, der gezielt jedes Mal eine neue Geräte-ID erzeugt — das war explizit gewünscht als "einfachste Lösung ohne Nebenwirkungen").

## Performance-Optimierungen (heute)

- `loadData()`: 4 unabhängige Ladeschritte (Affiliate-Agenten, Site-Settings, Kategorien/Styles, erste Produktseite) liefen nacheinander → jetzt parallel per `Promise.all` (mit `Promise.resolve()`-Wrapper, da der Supabase-Query-Builder nur "thenable" ist, kein echtes Promise mit `.catch()`).
- `loadRemainingProducts()` (Hintergrund-Nachladen bei >300 Produkten): lief sequentiell Seite für Seite → jetzt 4 Seiten gleichzeitig, Ergebnisse werden trotzdem in korrekter Reihenfolge angehängt.
- Kategorien/Styles/Subcats werden jetzt 10 Minuten pro Creator in `localStorage` gecacht (ändern sich selten).
- `<link rel="preconnect">` zu Supabase und cdn.jsdelivr.net ergänzt.
- Bild-Komprimierung beim Upload war schon vorher gut gelöst (max. 1000px, JPEG Qualität 0.82, PNG bleibt PNG wegen Transparenz).

Gemessen mit einem temporären Test-Creator (5000 Produkte, danach sauber wieder gelöscht): erste sichtbare Produkte nach ~1,8-2,2s (jetzt schneller durch obige Parallelisierung), Rest lädt im Hintergrund nach, blockiert die Seite nicht.

## Bekannte offene Punkte (keine Bugs, nur zur Kenntnis)

- Admin-Fake-E-Mail-Kopplung Frontend/Backend ist aktuell synchron, aber fragil bei künftigen Änderungen (siehe oben).
- Zwei tote OAuth-Edge-Functions, nicht löschbar über verfügbare Tools, harmlos.
- Storage-Bucket "produktbilder" ist ungenutzt (nur "products" wird tatsächlich verwendet) — Aufräumpunkt, kein Risiko.
- Supabase-Security-Advisor meldet mehrere SECURITY-DEFINER-Funktionen als "von anon aufrufbar" — individuell geprüft: alle prüfen intern selbst `is_admin()`/`can_manage_creator()`, keine echte Lücke.
- "Leaked Password Protection" bei Supabase Auth war deaktiviert — liegt in der Verantwortung des Betreibers (Dashboard-Einstellung, kein Code-Fix möglich).
- Shop-Zuweisung: bewusst auf "bis zu 3 Creator teilen sich einen Shop" ausgelegt (Kommentar im Code bestätigt das) — es gibt bereits eine vollständige Übersichtsliste aller bestehenden Zuweisungen ("Zugewiesene Gruppen"), keine fehlende Funktion.

## Wichtige Dateien/Stellen im Code (Zeilennummern können sich verschieben)

- `CREATOR`-Objekt, `CATEGORIES`, `STYLES` — Fallback-Werte für die Demo-Ansicht ohne `?creator=`
- `loadData()` — läd alles für die Storefront
- `buildProductCard()` — Produktkarten-Rendering
- `renderDashboard()`, `renderAnalytics()`, `renderRevenue()`, `renderShopAssign()` — Admin-Ansichten
- `wireDashboard()` — verdrahtet alle Admin-Formulare/Buttons
- `boot()` — Einstiegspunkt ganz am Dateiende

## Hinweis zu dieser Übergabe

Es gab in der vorherigen Sitzung eine längere Diskussion über eine geplante Konto-Migration (neues Supabase-/Claude-Konto), die auf Wunsch des Nutzers **nicht weiterverfolgt** wurde — dazu wurden keine Zugangsdaten oder Exporte erstellt oder verschickt. Falls das Thema in dieser neuen Sitzung wieder aufkommt: es ist offen, nichts wurde vorbereitet.
