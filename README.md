# गीता प्रेस — Post-Press System

## आज का ढाँचा

| परत | क्या |
|---|---|
| Database | Supabase (PostgreSQL), project `agntdlbaspqjwaiyoxnp`, schema `gp` |
| API | Supabase Data API (PostgREST) — `gp` schema exposed है |
| Frontend | `supabase/` की 8 static HTML files, GitHub Pages पर |
| Auth | Supabase Auth, email + password |

Google Sheet और Apps Script **अब इस्तेमाल में नहीं** हैं। पुरानी files backup
के तौर पर रखी हैं।

## Files

`supabase/` folder — यही GitHub पर जाता है:

Site खोलने पर **Dashboard** आता है — वही `index.html` है.

| File | कहाँ लिखता है |
|---|---|
| `index.html` | **Dashboard** — आठों tables का date-wise Excel export |
| `main_entry.html` | Main Entry — contractor `By Machine` या `Channel` हो तो `by_machine`, वरना `main_entry` |
| `soft_bound.html` | Binding Type से `perfect_binding` या `center_pinning` |
| `laghu_ati_laghu.html` | `laghu_ati_laghu` |
| `stm.html` | `stm` |
| `process_entry.html` | `process_entry` — rate master से rate और divisor |
| `print_order.html` | `print_order` — पिछले edition की history database से |
| `reconcile.html` | पढ़ता है `v_reconcile` view |

SQL files:

| File | कब |
|---|---|
| `supabase_schema.sql` | एक बार, नया project बनाते वक़्त |
| `supabase_auth_rls.sql` | schema के बाद, **पहला user बना लेने पर** |
| `seed/*.sql` | masters भरने के लिए — books, contractors, rates, divisors |
| `supabase_cancel.sql` | रद्द करने की व्यवस्था — columns, policies, `v_cancelled` |
| `supabase_books_flags.sql` | `books.panni` / `books.box` के झंडे |
| `supabase_print_order_machine.sql` | `print_order.machine_type` |
| `supabase_clear_trial_data.sql` | ट्रायल data मिटाने के लिए (⚠️ वापस नहीं आता) |
| `supabase_view_fix.sql` | `v_reconcile` का सुधार (schema में शामिल है, अलग से ज़रूरत नहीं) |

## रोज़मर्रा के काम

### नई किताब

Supabase → **Table Editor** → `gp.books` → Insert row:
`code`, `name`, `juje`, `size`, `page_type`

GitHub पर कुछ नहीं करना — form अगली बार उसी code पर उठा लेगा।

`size` वही value होनी चाहिए जो `process_rate` में है (`पुस्तकाकार`,
`ग्रन्थाकार`, `पाकेट`, `लघु आकार` आदि), वरना Process Entry में उस किताब के
लिए कोई process नहीं दिखेगा।

### Rate बदलना

`gp.process_rate` में वो row बदलें। Key है
`process + size + machine + special_option + juje_range`.

`juje_range` में `*` का मतलब "किसी भी juje पर लागू" (flat rate)।
अंकों वाली range हो तो `juje_from` और `juje_to` भी सही करें — form इन्हीं
से मिलान करता है, `juje_range` के text से नहीं।

### नया contractor

`gp.contractors` में row: `code`, `group_name` — इससे **Soft Bound** में अपने
आप आ जाएगा.

बाकी चार forms (`main_entry`, `stm`, `laghu_ati_laghu`, `process_entry`) में
सूची HTML में लिखी है, तो उनमें `<option>` हाथ से जोड़ना पड़ता है. चारों की
सूची एक जैसी रखें.

`By Machine` group में कोई नया मशीन विकल्प जोड़ें तो `main_entry.html` में
`BY_MACHINE` सूची में भी उसका नाम डालें — वही तय करती है कि row `by_machine`
table में जाएगी और पाँच fields अपने आप भरेंगे.

### नया operator

Authentication → **Users → Add user**
- Email: `naam@gitapress.org` — असली inbox ज़रूरी नहीं, format सही चाहिए
- Password
- **Auto Confirm User** ✓ — यह छूटा तो login नहीं होगा
- User Metadata: `{"full_name": "नाम"}`

### Password

**बदलना** — form में ऊपर `Password` button (login किए हुए)।

**भूल जाना** — खुद से नहीं बनता। recover email भेजता है और
`@gitapress.org` वाले पते असली नहीं हैं। व्यवस्थापक Authentication → Users
से नया password देगा।

## जो जानना ज़रूरी है

### S.N. database देता है

`id` column `generated always as identity` है। दो operator एक ही पल में save
करें तो भी अलग-अलग नंबर मिलेंगे। पहले Apps Script में `getLastRow()` से नंबर
निकलता था और यह टकराव सच में होता था।

### entered_by भरोसे की चीज़ नहीं

वो login किए खाते से भरता है, पर असली पहचान `user_id` में है — जो database
`auth.uid()` से खुद डालता है और browser से बदली नहीं जा सकती। हिसाब में
`user_id` देखें।

### गलत entry रद्द करना

**आज की, अपनी entry** — form में नीचे **आज की मेरी entries** card, उसमें
`रद्द` button. कारण लिखना ज़रूरी है.

**पुरानी entry** — SQL Editor से. uuid कभी हाथ से न लिखें, email से उठाएँ:

```sql
update gp.process_entry
set cancelled     = true,
    cancelled_at  = now(),
    cancelled_by  = (select id from auth.users where email = 'jitendra@gitapress.org'),
    cancel_reason = 'गलत quantity — व्यवस्थापक ने रद्द किया'
where id = 27;
```

रद्द होने के बाद row मिटती नहीं — निशान लगा रहता है, reconcile और dashboard
उसे गिनना छोड़ देते हैं. सारी रद्द entries एक जगह:

```sql
select * from gp.v_cancelled order by cancelled_at desc;
```

operator की सीमाएँ (जान-बूझकर): अपनी ही entry, उसी दिन की, एक बार रद्द तो
दोबारा नहीं, और कारण के बिना नहीं. `quantity`/`rate` वो छू ही नहीं सकता —
UPDATE की अनुमति सिर्फ़ चार cancel columns पर है.

### edit या delete नहीं हो सकता

किसी value को बदला नहीं जा सकता — केवल रद्द किया जा सकता है (ऊपर देखें).
Session table की row पर click करने से **edit नहीं होता** — वो उसी किताब के
लिए form भर देता है ताकि अगली entry जल्दी हो.

सचमुच कोई value सुधारनी हो तो Supabase Table Editor से, जहाँ dashboard login
चाहिए. ध्यान रखें: `quantity` बदलने पर `total_amount` अपने आप नहीं बदलता, और
Print Order में `total_quantity_printed` भी नहीं. आम तौर पर पुरानी entry रद्द
करके नई करना ज़्यादा साफ़ रहता है.

### Reconcile का नियम

Print Order की मात्रा benchmark है। **STM और Process Entry दो अलग चरण हैं —
जोड़े नहीं जाते।** हर एक अपने आप में Print Order के बराबर होना चाहिए।

Process Entry में सिर्फ़ **अंतिम चरण** के तीन processes गिने जाते हैं:
`कटिंग(अजिल्द)`, `सजिल्द तैयारी`, `सजिल्द तैयारी (delux)`. इनका जोड़ सही है
क्योंकि किताब या अजिल्द रास्ते से जाती है या सजिल्द। बीच के processes (तार
सिलाई, मिसिल, निपिंग) नहीं गिने जाते — वरना हर copy कई बार गिनी जाती।

Edition का मिलान `edition_key()` से अंक निकालकर होता है, तो `2`, `2nd` और
`II संस्करण` एक ही edition माने जाते हैं।

### Total का हिसाब

Process Entry में `Total = rate × quantity ÷ rate_per`.
`rate_per` `process_per_count` से आता है — 14 processes पर 100, 12 पर 1000.
यह भूलने पर आँकड़ा 100 या 1000 गुना बड़ा दिखता है।

## बाकी काम

**`perfect_binding` और `center_pinning` की बनावट।** इनमें
process/wages/contractor के सात जोड़े columns में फैले हैं — यह Google Sheet
की नकल है, database की सही बनावट नहीं। वो सात rows होनी चाहिए, जैसे
`process_entry` में हैं। आज कुछ टूटता नहीं, पर "किस process पर कितना काम
हुआ" पूछना मुश्किल रहेगा, जबकि `process_entry` में आसान है। ठीक करने पर
`soft_bound.html` दोबारा बनाना पड़ेगा।

**Book Size की दो सूचियाँ।** `books.size` में 14 values हैं (`पत्रिका` और
उसके रूप शामिल), पुरानी `book_Size.xlsx` में 10 थीं। Rate master 14 वाली
सूची से चलता है। 136 किताबों का size खाली है — उन पर Process Entry rate
नहीं निकाल पाएगा और साफ़ message दिखाएगा।

**Rate master की दो अनसुलझी बातें।** मूल Excel में 44 rate keys पर 2-vs-1
का टकराव था; बहुमत वाला लिया गया। और 17 जगह juje ranges एक-दूसरे पर चढ़ती
हैं — form सबसे संकरी चुनता है, पर बाकी विकल्प dropdown में दिखते हैं ताकि
operator बदल सके।

## Backup

Supabase free tier में रोज़ का automatic backup नहीं है। महीने में एक बार
Dashboard से आठों tables का Excel निकाल लें — दोनों dates खाली छोड़ें तो
पूरा data आ जाता है।
