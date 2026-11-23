# 🛡️ Rasd (رصد) — Early Detection of Fraudulent Links and Websites

**ScamShield AI Hackathon — Abu Dhabi Police × 42 Abu Dhabi**
**Team:** Falcon Eye (فريق عين الصقر) — Abu Dhabi University

Rasd is a working MVP that finds fraudulent websites impersonating UAE government entities, banks and delivery companies, often **before** victims report them. One button discovers suspicious links from four sources, analyses every site with 50+ explainable indicators, gives it a risk score from 0 to 100 with reasons, and exports an evidence report.

> 📄 Full technical documentation (architecture, model choices, privacy design): [`docs/TECHNICAL_DOCUMENTATION.md`](docs/TECHNICAL_DOCUMENTATION.md)
> 🖥️ Presentation: [`docs/Rasd_Presentation.pptx`](docs/Rasd_Presentation.pptx) · [`docs/Rasd_Presentation.pdf`](docs/Rasd_Presentation.pdf)
> 📑 Documentation PDF: [`docs/TECHNICAL_DOCUMENTATION.pdf`](docs/TECHNICAL_DOCUMENTATION.pdf)
> 🇦🇪 Arabic summary at the end of this file — [ملخص بالعربية](#ملخص-بالعربية)

---

## Table of contents
1. [The problem](#1-the-problem)
2. [The solution](#2-the-solution)
3. [How to run it](#3-how-to-run-it)
4. [Reasoning behind our technical choices](#4-reasoning-behind-our-technical-choices)
5. [Trade-offs we made](#5-trade-offs-we-made)
6. [What we left out](#6-what-we-left-out)
7. [What we would do differently with more time](#7-what-we-would-do-differently-with-more-time)
8. [Results from a real run](#8-results-from-a-real-run)
9. [Testing](#9-testing)
10. [Project structure](#10-project-structure)
11. [Originality and AI assistance](#11-originality-and-ai-assistance)

---

## 1. The problem

Fraudsters build websites that copy the names and logos of official UAE entities (police fines, government services, banks, parcel delivery) to steal card details, OTP codes and personal data. Three properties make them hard to stop:

- **They impersonate trusted entities.** Abu Dhabi Police has repeatedly warned about fake sites using names and logos similar to official bodies.
- **They reach victims through search results, ads and map listings**, not only through SMS.
- **They live for a short time.** Sites are often set up briefly and disappear once money is taken, so a response that starts *after* a victim's report is usually too late.

Most defences are reactive: wait for a report, verify, then request a takedown. The hackathon challenge asks for the opposite: **early identification** of fraudulent links and websites across the open internet, including search engines and maps.

## 2. The solution

Rasd turns the process around: instead of waiting for victims, it actively looks for the fake site first.

**One click runs the full pipeline:**

| Stage | What happens |
|---|---|
| 1. Certificate Transparency | Queries public CT logs (crt.sh) for HTTPS certificates issued in the last 30 days to domains containing protected names (e.g. `adpolice`, `salik`, `uaepass`). A phishing site usually needs a certificate before it goes live, so this often catches it before its links spread. |
| 2. Look-alike domains | Generates typosquatting / homoglyph variants of each official domain (`adpo1ice.com`, `salik.co`, `uaepas.com`, `tamm-pay.online` …) and checks in DNS which ones actually exist. |
| 3. Search engines & ads | Runs the queries scammers target (e.g. "دفع مخالفات شرطة أبوظبي", "Salik fine payment") and collects organic results and paid ads. |
| 4. Google Maps | Scans business listings and flags listings that carry an official name but link to an unofficial website. |
| 5. Manual / SMS input | An analyst can paste any link or a full SMS message; links are extracted automatically. |
| 6. Deep analysis | Each link is analysed in four layers: **URL** (impersonation, typosquatting, cheap TLDs, shorteners), **domain** (age via RDAP, TLS certificate, redirects, hosting), **page content** (card/CVV/OTP fields, forms posting elsewhere, Telegram-bot exfiltration, hot-linked official logos, Arabic/English urgency language) and **optional intelligence** (Google Safe Browsing, VirusTotal, Claude AI review, ML model). |
| 7. Explainable score | All indicators are combined into a 0–100 risk score. Every point is backed by a reason the analyst can verify. |
| 8. Evidence report | Printable report (save as PDF from the browser) and CSV with IP addresses, registrar, registration date, first-seen time and defanged links (`hxxps://evil[.]com`). |

Verdicts: **70–100** likely fraud · **40–69** suspicious, needs review · **0–39** low risk · **official** (known official domain).

The dashboard is bilingual (Arabic / English), keeps a local history of scans, and runs on any laptop.

## 3. How to run it

**Requirement:** Python 3.9 or newer — <https://www.python.org/downloads/>. There is **no build step and nothing to `pip install`**: Rasd uses only the Python standard library.

| OS | Run |
|---|---|
| Windows | Double-click `START_WINDOWS.bat` (when installing Python, tick **"Add python.exe to PATH"**). |
| macOS | Double-click `START_MAC.command` (first time: right-click → Open). If you see certificate errors, run `Install Certificates.command` from the Python folder in Applications once. |
| Linux | `./start_linux.sh` |
| Any | `python app.py` (options: `--port 8765`, `--no-browser`) |

The dashboard opens automatically at <http://127.0.0.1:8765>. Then:

1. Press **"ابدأ الفحص الشامل / Start full scan"**, or
2. Paste links or a whole SMS message in the text box and press **"فحص هذه الروابط فقط / Check these links only"**.
3. Click any row to see every reason and the technical evidence; export with **PDF report** or **CSV**.

**Optional API keys** (the core works without any key). Add them in the dashboard's **Settings**, or as environment variables:

| Key | Adds | Environment variable |
|---|---|---|
| SerpApi | Google results, Google ads, Google Maps listings (free plan: 250 searches/month; one full scan uses ~10) | `RASD_SERPAPI_KEY` |
| Google Safe Browsing | Google's list of dangerous sites | `RASD_GSB_KEY` |
| VirusTotal | 70+ security engines (riskiest 4 domains per scan) | `RASD_VT_KEY` |
| Anthropic (Claude) | AI review of suspicious page content | `RASD_ANTHROPIC_KEY` (`RASD_ANTHROPIC_MODEL` optional) |

Keys are stored only on the local machine in `settings.local.json`, which is excluded from git.

**Customisation:** `config.json` holds the protected brands (names, official domains, keywords), search and Maps queries, the allowlist and scan limits. New entities can be added without code changes.

**Optional ML layer:** `pip install pandas scikit-learn`, then
`python tools/train_model.py <dataset.csv> --url-col URL --label-col label --phishing-value <value>`
writes `model_weights.json`, which the app then uses with no extra packages. Check the printed samples to confirm which label value means "phishing" in your dataset.

## 4. Reasoning behind our technical choices

- **Python standard library only (zero dependencies).** Police analysts and judges must be able to run the tool on any laptop with a double-click, offline from package registries and without dependency conflicts. The web server (`http.server`), HTTP client (`urllib`), HTML parser (`html.parser`), TLS inspection (`ssl`) and storage (`sqlite3`) are all built in.
- **Explainable, evidence-based scoring instead of a black-box model.** Law enforcement needs to justify every action, and no labelled UAE-specific dataset was available. Each indicator has a weight between 0 and 1 and a human-readable reason in Arabic and English. Indicators are combined with a *noisy-OR*: `risk = 1 − Π(1 − wᵢ)`, so several medium signals add up without any single weak signal dominating; trust signals (official domain, allowlist, long-established domain, redirect to the official site) reduce the score.
- **Discovery before analysis.** The challenge is about *early* identification, so we invested in sources that see a site before victims do: Certificate Transparency logs and DNS-checked look-alike domains, plus search ads and Maps listings that the challenge explicitly mentions.
- **Brand impersonation logic tuned for short names.** We use Damerau-Levenshtein distance, homoglyph normalisation (`0→o`, `1→l`, `rn→m` …) and token rules for short brand names (e.g. `tamm`, `adcb`) to avoid flagging unrelated words like `tammy-shop.com`.
- **Static page analysis.** Pages are downloaded as HTML (max 2 MB, 10 s timeout) and parsed without executing JavaScript. This is fast enough to analyse 200 sites in under 5 minutes and keeps the analyst's machine safe.
- **Optional external intelligence behind keys.** Google Safe Browsing and VirusTotal give strong confirmation when available; Claude reviews only pages that already score ≥ 25, to limit cost and data sharing. AI output is one weighted signal (at most 0.6 × confidence), never the sole decision.
- **Optional logistic-regression model.** Interpretable coefficients and pure-Python inference (no NumPy at runtime). It is trained by the user on a public dataset rather than shipped, because we did not have a UAE-labelled dataset to validate a shipped model.
- **Local web dashboard.** A browser UI is familiar to analysts, works on Windows/macOS/Linux, and the server binds to `127.0.0.1` only.

## 5. Trade-offs we made

| We chose | Instead of | Why, and the cost |
|---|---|---|
| Static HTML fetch | Headless browser rendering | Fast and safe; but phishing kits that build the page with JavaScript, or that cloak content from bots, can hide content-level indicators. URL/domain/discovery signals still apply. |
| Hand-set indicator weights | A trained classifier | Explainable and works without labelled data; weights need calibration with real confirmed cases to reduce false positives. |
| Built-in public-suffix list | `tldextract` (full list) | No dependency; rare multi-part suffixes may be parsed imperfectly. |
| SerpApi for Google & Maps | Scraping Google directly | Reliable and within terms; needs a key (free tier is enough for the hackathon). Without a key Rasd falls back to DuckDuckGo/Bing HTML, which can be rate-limited, and skips Maps. |
| Public crt.sh service | Running our own CT log monitor | Zero infrastructure; crt.sh can be slow or busy, so each keyword query is time-limited and failures are logged and skipped. |
| Cap of 200 analysed links per scan, 60 look-alikes per brand | Unlimited | Keeps a full scan under ~5 minutes; lower-priority links beyond the cap are not analysed (configurable in `config.json`). |
| Single-user local app | Multi-user server | Simple to run and keeps investigation data on the analyst's machine; no authentication or shared case management yet. |

## 6. What we left out

- **Visual similarity** (screenshot comparison with official sites).
- **Real-time CT streaming** (CertStream); we query CT logs per scan instead.
- **Detection of parked domains and defensive registrations.** Our live run showed this matters: some domains that carry a brand name with another extension may belong to the brand itself. Today these are flagged for human review.
- **A shipped, trained ML model** (no UAE-labelled dataset to validate it).
- **WHOIS for ccTLDs without RDAP** (e.g. some `.ae` domains show no registration date).
- **Social-media and messaging-platform ads** (Instagram, WhatsApp, Telegram channels).
- **Authentication, roles, audit log and automatic takedown requests.**

## 7. What we would do differently with more time

1. **Calibrate with real cases:** build a labelled dataset from confirmed Abu Dhabi Police cases, tune weights and train a gradient-boosted model with per-feature explanations (e.g. SHAP) to keep explainability.
2. **Sandboxed rendering:** a headless browser in an isolated container to execute JavaScript safely, capture screenshots and compare them with official sites (perceptual hashing).
3. **Reduce false positives:** detect parked pages and defensive registrations (registrant matches the brand, redirect to the official site, parking-provider name servers).
4. **Operational deployment:** continuous monitoring (CertStream + scheduled scans), a job queue, multi-user access with authentication and audit trail, and automatic takedown requests to registrars and hosting providers.
5. **Analyst OPSEC:** route page fetches through a dedicated egress proxy/VPN so scam operators cannot see or cloak against the investigator's IP.

## 8. Results from a real run

A full live scan was run on **22 November 2026** with SerpApi enabled:

| Metric | Value |
|---|---|
| Links discovered in one scan | **226** |
| Links analysed in depth (top priority, cap 200) | **200** |
| Classified "likely fraud" (70–100) | **23** |
| Classified "suspicious — needs review" (40–69) | **108** |
| Classified "low risk / official" (0–39) | **69** |
| Total time for discovery + analysis | **297 seconds** (under 5 minutes) |

These are automated estimates. **Every flagged site needs human verification before any action**; in particular, some look-alike domains may be defensive registrations owned by the brand itself.

## 9. Testing

```bash
python -m unittest discover -s tests -v
```

23 automated tests cover URL features and brand impersonation, typosquat generation, page-content analysis, end-to-end analysis with simulated network responses, scoring maths, deduplication, report escaping/defanging, the optional ML layer, and the web server (dashboard, scan flow, reports, cross-origin protection, path-traversal protection).

## 10. Project structure

```
app.py                     Starts the local server and opens the browser
config.json                Protected brands, search/Maps queries, allowlist, scan limits
START_WINDOWS.bat          Double-click launcher (Windows)
START_MAC.command          Double-click launcher (macOS)
start_linux.sh             Launcher (Linux)
rasd/
  pipeline.py              One-click scan: discover → deduplicate → blocklists → analyse → rank → save
  sources.py               Discovery: CT logs, look-alike DNS sweep, search engines, Google Maps
  features.py              URL features, brand impersonation, typosquatting, homoglyphs
  netcheck.py              DNS, domain age (RDAP), TLS certificate, page download
  content.py               Static HTML analysis of the page
  analyzer.py              Per-URL analysis, incl. internal-network guard
  intel.py                 Google Safe Browsing, VirusTotal, Claude review (optional)
  scoring.py               Noisy-OR score, verdicts, optional ML model
  storage.py               SQLite history (data/rasd.db)
  report.py                Printable HTML (→ PDF) and CSV evidence reports
  server.py                Local JSON API + static files
  util.py, config.py, signals.py
static/                    Bilingual dashboard (HTML/CSS/JS, no build step)
tools/train_model.py       Optional ML training
tests/test_rasd.py         Test suite
docs/                      Technical documentation (MD + PDF) and presentation (PPTX + PDF)
```

## 11. Originality and AI assistance

Rasd was built for this hackathon by team Falcon Eye. It contains no third-party source code; it calls public services (crt.sh, rdap.org, DNS) and optional APIs only through their documented interfaces. Brand names and official domains in `config.json` are used solely to detect impersonation.

Parts of the code and documentation were written with the help of an AI assistant (Claude by Anthropic). The team ran, tested and evaluated the system, including the live scan reported above.

---

## ملخص بالعربية

**المشكلة:** مواقع مزيّفة تنتحل أسماء وشعارات الجهات الرسمية والبنوك وشركات الشحن في الإمارات، وتصل إلى الضحايا عبر نتائج البحث والإعلانات والخرائط والرسائل، وتعمل لوقت قصير ثم تختفي. لذلك فالاستجابة بعد البلاغ تأتي متأخرة غالباً.

**الحل — «رصد»:** بضغطة زر واحدة يكتشف المواقع المشبوهة من سجلات شهادات الأمان، والنطاقات المشابهة للمواقع الرسمية، ونتائج البحث وإعلاناته، وخرائط Google، والروابط ورسائل SMS التي يلصقها المحلل. ثم يحلّل كل موقع بأكثر من 50 مؤشراً في أربع طبقات (الرابط، النطاق، محتوى الصفحة، الاستخبارات الاختيارية)، ويعطيه درجة خطورة من 0 إلى 100 مع سبب واضح لكل نقطة، ويُصدر تقرير أدلة بروابط مُعطَّلة.

**التشغيل:** يحتاج فقط Python 3.9 أو أحدث، دون تثبيت أي مكتبات. انقر مرتين على `START_WINDOWS.bat` (ويندوز) أو `START_MAC.command` (ماك)، أو نفّذ `python app.py`. تفتح الواجهة على `http://127.0.0.1:8765`. المفاتيح الاختيارية (SerpApi وSafe Browsing وVirusTotal وClaude) تُضاف من زر «الإعدادات» وتبقى على الجهاز فقط.

**أسباب الاختيارات التقنية:** مكتبات Python القياسية فقط ليعمل على أي جهاز بنقرة مزدوجة. درجة قابلة للتفسير بدل نموذج مغلق، لأن جهات إنفاذ القانون تحتاج تبرير كل إجراء، ولعدم توفر بيانات إماراتية مصنّفة. تحليل ساكن للصفحة دون تنفيذ JavaScript لسرعة الفحص وحماية جهاز المحلل.

**المفاضلات:** الجلب الساكن أسرع وأكثر أماناً لكنه قد يفوّت صفحات تُبنى بالكامل عبر JavaScript. الأوزان اليدوية قابلة للتفسير لكنها تحتاج معايرة ببلاغات مؤكدة. SerpApi طريقة نظامية لنتائج Google والخرائط لكنها تحتاج مفتاحاً (الخطة المجانية تكفي). حدّ 200 رابط لكل فحص ليبقى الفحص تحت 5 دقائق.

**ما تركناه:** المقارنة البصرية، والمراقبة اللحظية للشهادات، وكشف النطاقات المركونة والمملوكة للجهة نفسها، ونموذج تعلم آلي مدرَّب جاهز، ومنصات التواصل الاجتماعي، والصلاحيات وطلبات الإغلاق التلقائية.

**ما سنفعله بوقت أطول:** معايرة النظام ببلاغات مؤكدة من شرطة أبوظبي، وتصيير معزول للصفحات مع لقطات شاشة، وتقليل الإنذارات الخاطئة، ونشره كخدمة مراقبة مستمرة متعددة المستخدمين، وإخفاء عنوان المحلل عن المحتالين أثناء الفحص.

**نتائج تشغيل حقيقي (22 نوفمبر 2026):** 226 رابطاً مكتشفاً، و200 رابط محلل بعمق، و23 «احتيالي على الأرجح»، و108 «مشبوه»، و69 «منخفض الخطورة / رسمي»، خلال 297 ثانية. هذه تقديرات آلية تحتاج مراجعة بشرية قبل أي إجراء.

**الاختبارات:** `python -m unittest discover -s tests -v` — ‏23 اختباراً آلياً ناجحاً.
