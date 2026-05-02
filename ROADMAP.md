# BarTapp — Roadmap

Documento di lavoro. Le feature sono raggruppate per fase logica, non per scadenza.
Le voci marcate **[utente]** vengono dalla richiesta originale; le **[+]** sono aggiunte
proposte che potrebbero essere determinanti per il successo dell'app.

---

## Fase 0 — Fondamenta multi-tenant

Prima di aggiungere feature, serve il modello dati che le supporta. Questa fase
è prerequisito di tutto il resto.

### 0.1 — Multi-locale e registrazione dispositivi **[utente 1]**

- Concetto di **Venue** (locale): nome, owner, configurazione branding (logo, colori).
- Concetto di **Event** (evento): appartiene a un venue, ha un menu di ticket type
  (vedi 2.1), date di inizio/fine, stato (draft / active / archived).
- **Device registration**: ogni dispositivo che apre l'app si registra al venue
  con un codice di pairing (6 cifre, valido 5 min, generato dal Master).
- Ruoli device:
  - `master` — può creare/modificare venue, eventi, gestire altri device
  - `generator` — può solo generare ticket nell'evento attivo
  - `burner` — può solo bruciare ticket
  - `generator+burner` — entrambi (default per chi ha le credenziali del master)
- **Revoca device**: il master può sganciare un device dal venue (kill switch).

### 0.2 — Differenziazione GENERATOR vs BURN **[utente 1.1]**

Già coperto dai ruoli sopra. Il master decide il ruolo al momento del pairing.
Lo stesso device fisico può avere ruoli diversi su eventi diversi.

### 0.3 — Modello dati Firestore aggiornato **[+]**

```
venues/{venueId}
  - name, ownerEmail, branding{logoUrl, primaryColor}, createdAt

venues/{venueId}/devices/{deviceId}
  - name, role, pairedAt, lastSeenAt, revokedAt

venues/{venueId}/events/{eventId}
  - name, status, startsAt, endsAt, menu[]

venues/{venueId}/events/{eventId}/tickets/{ticketId}
  - (campi attuali) + ticketTypeId
```

Migrazione: tool una-tantum che sposta i ticket esistenti in
`venues/legacy/events/legacy/tickets/*` per non perdere dati storici.

---

## Fase 1 — Sicurezza **[utente 2]**

Stato attuale: regole Firestore aperte (`allow read, write: if true`),
sicurezza by-obscurity dell'URL. Funziona per evento privato singolo,
non scala a multi-locale.

### 1.1 — Firebase Authentication
- **Anonymous auth** per i buyer (Fase 3): zero attrito, ogni utente ha un UID.
- **Email link auth** (passwordless) per il master del venue.
- **Custom claims** sul JWT per ruolo device (master/generator/burner).
- **Contro**: aggiunge ~200ms al cold start; richiede gestione del token refresh
  su Safari iOS che è notoriamente fragile (cookie SameSite, ITP).

### 1.2 — Firestore Security Rules granulari
- Lettura ticket: solo device del venue.
- Scrittura ticket: solo `generator` o `master`.
- Bruciatura (transizione `usedAt: null → timestamp`): solo `burner` o `master`.
- Validazione formato ID, lunghezza campi, no campi extra.
- **Contro**: debug più difficile (rules vanno testate con emulatore);
  qualunque bug nelle rules può rompere l'app in produzione.

### 1.3 — App Check
- Blocca chiamate Firestore da client non autorizzati (no curl, no script).
- Su iOS usa DeviceCheck/AppAttest, su web reCAPTCHA v3.
- **Contro**: reCAPTCHA aggiunge una richiesta esterna (lentezza, privacy);
  può dare falsi positivi su iPhone vecchi o reti pessime al bar.

### 1.4 — Cifratura ID ticket **[+]**
- Gli ID attuali (`DT-XXXXXXXX`) sono ~40 bit di entropia.
  Brute force teoricamente possibile (1B tentativi). Con regole aperte è un rischio.
- Aggiungere HMAC firmato lato server (Cloud Function) verificato al burn.
- **Contro**: serve una Cloud Function → uscita dal free tier se il volume cresce.

### 1.5 — Rate limiting **[+]**
- Cloud Function che limita: max N burn/min per device, max N generate/min per event.
- Protegge da device compromesso che brucia tutto.
- **Contro**: complessità + costo Cloud Functions; latenza extra sul burn (critica al bar).

### 1.6 — Audit log immutabile **[+]**
- Sub-collection `auditLog` write-only (rules: `allow create; allow update, delete: if false`).
- Ogni burn/generate logga: deviceId, ticketId, timestamp, IP (da Cloud Function).
- **Contro**: raddoppia le scritture Firestore → costi.

### Decisione consigliata
Fase 1.1 + 1.2 + 1.3 sono il minimo per multi-locale. 1.4–1.6 solo se l'app
viene usata per eventi commerciali con biglietti pagati (Fase 3).

---

## Fase 2 — Eventi configurabili **[utente 3]**

### 2.1 — Definizione evento con menu di ticket type
- Master crea evento: nome, data, location.
- Definisce **ticket types** liberamente (no preset rigidi):
  - Nome (es. "Drink", "Panino + birra", "Menu completo")
  - Icona/emoji
  - Prezzo (opzionale, per Fase 3)
  - Stock massimo (opzionale, es. "max 200 panini")
  - Colore di sfondo del ticket
- I generator entrano nell'evento attivo e selezionano il tipo dal menu.
  Niente più "scrivi nome evento ogni volta".
- **[+]** Template evento riutilizzabile: "Sagra di paese" → preset salvato e
  ricaricabile per l'edizione successiva.

### 2.2 — Switch evento attivo **[+]**
- Un device può avere più eventi attivi se il venue ne ha (es. due bar
  contemporanei nello stesso locale). Picker in alto.

### 2.3 — Fine evento + archivio **[+]**
- Bottone "Chiudi evento": blocca generazione, mantiene burn attivo per code residue,
  poi archivia. I ticket archiviati restano consultabili ma non bruciabili.
- Reportistica automatica al closing: venduti/usati/no-show, picco orario, revenue.

---

## Fase 3 — Pagamenti automatici e Buyer App **[utente 4]**

### 3.1 — Buyer App (terza app)
- URL pubblico: `bartapp/buy/{eventId}` o QR sul volantino dell'evento.
- Mostra il menu dell'evento con prezzi.
- Carrello → checkout → pagamento → ticket consegnato.
- **Niente** generazione manuale, lista ticket, conta, gestione.
- I ticket comprati vivono in localStorage del buyer + email + (opzionale) aggiunta a Wallet.

### 3.2 — Integrazione pagamento
- **Stripe Checkout**: link hosted, 1.5%+€0.25 in EU, supporta Apple/Google Pay.
  Webhook → Cloud Function → crea ticket → email.
- **PayPal**: simile ma webhook meno affidabile, UX checkout più lenta.
- **[+]** **SumUp/Satispay** per il mercato italiano: commissioni più basse,
  riconoscimento del brand al sud Italia.
- **Contro**: serve P.IVA / business account; serve Cloud Function (no free tier puro);
  serve gestione rimborsi e dispute.

### 3.3 — Email delivery
- **Resend** o **SendGrid free tier** (100 mail/giorno gratis).
- Email contiene: QR ticket inline, "Aggiungi a Apple/Google Wallet", link "I miei ticket".
- **[+]** **Apple Wallet (.pkpass)** e **Google Wallet** pass: enorme upgrade UX,
  i clienti non perdono il ticket nel rullino. Richiede certificato Apple Developer (€99/anno).

### 3.4 — Pagina "I miei ticket" per il buyer **[+]**
- Lista dei ticket comprati, stato (valido/usato), QR a tutto schermo al tap.
- Funziona offline (Service Worker + cache QR).
- Permette di trasferire un ticket a un amico (genera link monouso).

### 3.5 — Refund e void **[+]**
- Master può rimborsare un ticket non usato → Stripe refund + ticket → stato `refunded`.
- Stato `refunded` ≠ `used`: il burn lo rifiuta esplicitamente con messaggio chiaro.

### 3.6 — Monetizzazione **[utente]**

> **Decisione corrente (vedi `pricing/index.html`)**: modello SaaS pay-per-event
> (€5 / €12 / €25 a evento, primi 30 ticket gratis). I pagamenti dei drink
> **non** passano da BarTapp: il venue continua a usare i suoi strumenti
> (PayPal, contanti, POS). Il modello platform-fee Stripe Connect descritto
> sotto resta come **opzione futura** se BarTapp vorrà evolvere in marketplace.

#### 3.6.a — Modello SaaS pay-per-event (attuale)

Vantaggi:
- Zero vincoli PSD2 / istituto di pagamento — vendiamo solo software.
- Niente Cloud Functions necessarie per partire (basta un Stripe Payment Link
  o un mailto + bonifico).
- Responsabilità chiara: BarTapp non è coinvolto nelle dispute sui drink.
- Cash flow prevedibile (incasso fisso a evento, indipendente dal volume).

Cosa serve per supportarlo lato app:
- **Concetto di "evento attivo"** con scadenza 48h dal pagamento (vedi Fase 2).
- **Limite ticket** per piano (100 / 300 / illimitato): contatore lato Firestore,
  blocco soft con avviso prima del cap, hard block al cap.
- **Free tier 30 ticket**: contatore per venue, blocca generazione al 31° finché
  non viene attivato un piano. Tracciato lato Firestore (non localStorage,
  altrimenti basta cancellare cache per ricominciare).
- **Stripe Payment Link** statico per ogni piano (Small/Medium/Large): genera URL
  hosted, niente integrazione Connect, niente webhook obbligatori.
  Per attivare il piano dopo pagamento: webhook `checkout.session.completed`
  → Cloud Function → set `venues/{id}/plan` + `eventQuotaRemaining`. Oppure
  sblocco manuale via console Firebase finché i volumi sono bassi.
- **Pannello master**: piano attivo, ticket residui, scadenza evento,
  bottone "rinnova/upgrade".

Rischi del modello:
- Ticket-cap rigidi creano frizione quando un evento sfora di poco.
  Mitigazione: vendita di "pacchetti extra" (es. +100 ticket = €X) in-app.
- Difficile spiegare cos'è "1 evento" se il locale fa serate consecutive
  (3 sere di sagra = 3 piani?). Definire policy chiara nelle FAQ.
- Concorrenza facile: chiunque può clonare il software. Il valore va
  costruito su semplicità + supporto + brand del venue (vedi 5.1).

#### 3.6.b — Platform fee (opzione futura)

Se in futuro BarTapp vorrà passare a marketplace dove il gestore del locale
incassa direttamente i drink e BarTapp trattiene una commissione: vedi sotto.
È il modello classico (Airbnb, Glovo).

#### Vincolo legale fondamentale
Trattenere soldi di terzi senza essere autorizzati = attività di
**istituto di pagamento** (regolata da Banca d'Italia / PSD2).
Per evitare l'autorizzazione bancaria si usa un PSP "Connect-style" che
fa da istituto regolato per noi. **Non si può evitare questo passaggio.**

#### Opzioni PSP

**Stripe Connect** (raccomandato)
- Tre tier:
  - `Standard`: il venue ha un account Stripe proprio, paga lui le fee Stripe,
    noi prendiamo `application_fee` sopra. Onboarding lento (KYC pieno),
    UI Stripe-branded. Setup minimo lato BarTapp.
  - `Express`: onboarding rapido (~5 min), UI co-branded, KYC gestito da Stripe,
    BarTapp controlla la dashboard. **Il sweet spot per BarTapp.**
  - `Custom`: full white-label, BarTapp è responsabile di KYC/compliance.
    Più complesso, richiede impegno legale serio.
- Fee Stripe EU: 1.5% + €0.25 carte EU, 2.5% + €0.25 non-EU.
  La nostra `application_fee` si somma sopra (es. +5% di nostro = totale ~6.5%).
- Payout automatico al venue su SEPA, T+2.
- **Contro**: anche `Express` richiede al venue di fornire codice fiscale,
  IBAN, documento. Attrito non zero. Stripe può freezare i fondi se sospetta
  frode → BarTapp deve gestire il customer support arrabbiato.

**Mollie Connect**
- Equivalente europeo, supporta meglio i metodi locali (Bancomat Pay, iDEAL, MyBank).
- Fee comparabili. UI più pulita per il mercato EU.
- **Contro**: ecosistema/documentazione meno ricco di Stripe; meno SDK.

**PayPal Marketplace**
- Esiste ma sconsigliato: split delayed (i fondi arrivano al partner dopo holding
  variabile), UX checkout lenta, dispute frequenti.

**SumUp / Satispay**
- Brand riconosciuti in Italia. **Non hanno API marketplace mature**:
  l'integrazione split-payment è limitata o richiede contratti business custom.
  Tenibili come metodo di pagamento aggiuntivo, non come spina dorsale.

#### Modelli di fee configurabili

Il pannello master del venue definisce il deal con BarTapp. Possibili modelli:

- **Percentuale fissa** (es. 3% su ogni ticket). Semplice, scala col fatturato.
- **Flat per ticket** (es. €0.10/ticket). Prevedibile per il venue, può essere
  insufficiente per la nostra struttura costi sui ticket low-price.
- **Ibrido** (es. 2% + €0.05). Standard nei marketplace.
- **Subscription** (es. €29/mese, fee 0%). Adatto a venue ricorrenti che fanno
  volumi alti — il venue preferisce costi fissi.
- **Freemium**: primi 50 ticket/evento gratis, poi fee. Acquisition tool.
- **Tiered**: fee scende all'aumentare del volume mensile.

Il deal va salvato sul documento `venues/{id}` come campo `feeAgreement`,
versionato con `effectiveFrom` per non riscrivere lo storico.

#### Flusso tecnico (Stripe Connect Express)

```
1. Venue master clicca "Attiva pagamenti" nel pannello
2. Cloud Function crea Stripe Account → restituisce onboarding link
3. Venue completa KYC su pagina Stripe-hosted (5 min)
4. Webhook account.updated → segna venue come "payments_enabled"
5. Buyer compra ticket sulla buyer app
6. Cloud Function crea PaymentIntent con:
     amount: €X
     application_fee_amount: nostra_fee
     transfer_data.destination: stripe_account_venue
7. Stripe addebita buyer, splitta automaticamente:
     - venue riceve X - fee_stripe - nostra_fee
     - BarTapp riceve nostra_fee
8. Webhook payment_intent.succeeded → genera ticket → email
```

#### Implicazioni amministrative

- Serve **P.IVA BarTapp** (ditta individuale o SRL) per emettere fattura sulla
  fee al venue. La fattura è un documento mensile riepilogativo
  (es. "Servizio piattaforma BarTapp - aprile 2026 - €X + IVA 22%").
- Stripe emette report fiscali mensili che alimentano la contabilità.
- **Contratto piattaforma** (T&Cs) firmato dal venue al primo onboarding:
  copre fee, responsabilità su dispute, GDPR (BarTapp è data processor).
- **GDPR**: dati buyer (email, pagamento) → BarTapp processor, Stripe sub-processor,
  venue controller. Privacy policy aggiornata.

#### Anti-frode (operativo)

- Limite ticket/buyer/evento (es. max 20) per evitare carding test.
- Stripe Radar attivo (incluso, ML anti-frode).
- Refund automatico se chargeback: il chargeback fee (€15) è sostenuto da BarTapp
  o dal venue? Va deciso e scritto in T&Cs. Standard: fee al venue, BarTapp
  trattiene una riserva del 2-5% per i primi 90 giorni.

#### Dashboard finanziaria per il venue **[+]**
- "Quanto hai incassato oggi/questo evento/questo mese": split tra lordo,
  fee Stripe, fee BarTapp, netto sul conto.
- Export CSV per il commercialista.
- Notifica push al payout SEPA.

---

## Fase 4 — Operatività al bar **[+]**

Cose che fanno la differenza tra "demo che funziona" e "app che usi davvero
durante una sagra a 500 persone in 2 ore".

### 4.1 — Modalità offline robusta
- Service Worker che cacha l'app (già pseudo-fatto da PWA).
- Coda di burn locali con sync quando torna la rete.
- **Contro**: aprire la finestra offline = aprire la finestra a doppio burn
  (due burner offline brucianoi lo stesso ticket). Mitigazione: marcatura
  "burn provvisorio" da confermare al sync.

### 4.2 — Torcia/flash
- API `ImageCapture.getPhotoCapabilities().torch` su Chrome Android.
- Su iOS Safari **non disponibile**: fallback "alza la luminosità".

### 4.3 — Suono di conferma
- Beep diverso per OK / GIÀ USATO / NON VALIDO.
- Audio contesto (Web Audio) sbloccato al primo tap.
- **Contro**: il primo tap deve essere reale, non programmatico.

### 4.4 — Filtri lista per dispositivo
- "Mostra solo ticket bruciati da questo device" / "creati da Marco".
- Utile per turni e responsabilità.

### 4.5 — Dashboard live per il master
- Tablet sul bancone del master: contatori real-time, ticket/min, alert
  ("scorta panini quasi finita").

### 4.6 — Bouncer mode **[+]**
- Variante del burner per controllo accesso (entry ticket vs consumo).
- Distingue ingressi da consumi nel report finale.

---

## Fase 5 — Crescita e retention **[+]**

### 5.1 — Branding per venue
- Logo + colori del locale sui ticket PNG/PDF e nella buyer app.
- Marketing gratuito per ogni evento.

### 5.2 — Analytics aggregato (anonimizzato)
- Quanti eventi, ticket totali, revenue totale per venue.
- Dashboard owner che mostra crescita mese su mese.

### 5.3 — Recensioni post-evento
- Email post-evento al buyer con NPS 1-domanda + link al locale.

### 5.4 — Riconoscimento clienti ricorrenti
- Stesso device/email che torna a un evento → "Bentornato".
- Apre la strada a fidelity (10° drink gratis).

### 5.5 — Multilingua
- Italiano + inglese minimo. Stringa centralizzata in JSON.
- Sblocca eventi con turisti / locali in zone bilingue.

### 5.6 — Conformità fiscale italiana
- Lo scontrino elettronico per le consumazioni: integrazione con cassa o
  esportazione corrispettivi giornalieri per il commercialista.
- **Determinante** per i locali strutturati che vorrebbero usare BarTapp
  come cassa secondaria.

---

## Debito tecnico esistente (pre-roadmap)

Cose già note nel codice attuale che andrebbero affrontate prima di scalare:

- **No `enableIndexedDbPersistence`**: ricontrolla se nelle versioni recenti del
  Firebase SDK il problema Safari è risolto. Se sì, riattivare per resilienza offline.
- **3 file HTML standalone**: a un certo punto duplicare CSS/init Firebase
  in 3 file diventa insostenibile. Valutare un mini bundler (esbuild) o
  estrazione di un `shared.js` includibile via `<script>`.
- **ID ticket 40 bit**: vedi 1.4. Per ora va bene, ma alza l'entropia se i
  ticket diventano monetari.
- **Cache busting manuale**: aggiungere `?v={hash}` sugli script o
  `<meta http-equiv="Cache-Control">` per evitare hard reload manuale post-deploy.
