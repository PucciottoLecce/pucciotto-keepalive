# pucciotto-keepalive

"Sala di controllo" che tiene svegli i servizi **Render Free** e il **DB
Supabase** di PuccioQuiz, usando **GitHub Actions** schedulati. Sostituisce
del tutto cron-job.org.

## Perché esiste

I servizi su **Render Free** si addormentano dopo ~15 min di inattività e il
**risveglio a freddo dura ~50 secondi**. Il piano gratuito di **cron-job.org**
ha un **timeout massimo di 30s**: staccava la connessione *prima* che Render
finisse di avviarsi → il servizio non si svegliava mai davvero (loop di avvii
annullati a metà), e la query di keep-alive su Supabase non partiva.

La soluzione qui è un `curl` che **aspetta fino a ~150–180s** (`--max-time`):
tiene aperta la connessione oltre il cold start, così Render completa l'avvio,
risponde, ed esegue la query. GitHub Actions ha **minuti illimitati sui repo
pubblici**, quindi è gratis e sostenibile.

> Questo repo è **pubblico apposta** (minuti Actions illimitati) e **non
> contiene alcun segreto**: i workflow chiamano solo URL pubblici.

## I workflow

| Workflow | Cosa fa | Quando | Avviso email |
|---|---|---|---|
| [`keepalive.yml`](.github/workflows/keepalive.yml) | Tiene sveglio **Order** (`/ping`) | ogni **5 min**, 09:00–00:59 Rome | no (silenzioso) |
| [`order-healthcheck.yml`](.github/workflows/order-healthcheck.yml) | Controlla che **Order** risponda | **1×/ora**, in fascia | **sì**, se Order è giù in orario |
| [`quiz-keepalive.yml`](.github/workflows/quiz-keepalive.yml) | Query reale su **Supabase** (via Quiz `/keep-db-alive`) | **1×/giorno** ~12:00 Rome | **sì**, se non riesce |

Tutti si possono lanciare a mano dal bottone **"Run workflow"** (evento
`workflow_dispatch`) per un test immediato.

### Order — keepalive + healthcheck

- **keepalive**: pinga `https://orderpucciotto.onrender.com/ping` ogni 5 min
  per tenerlo caldo. Sui singoli errori **non fa rumore** (altrimenti, in un
  guasto, decine di mail).
- **order-healthcheck**: una volta l'ora verifica che Order risponda; se dopo
  i tentativi non risponde **in orario di apertura**, il job fallisce e parte
  **una** mail. Al massimo ~1 mail/ora durante un guasto reale.

### Quiz / Supabase

- **Supabase Free** mette in pausa il progetto dopo **~7 giorni** di
  inattività. L'endpoint `/keep-db-alive` esegue una **query vera**
  (`select id from quiz_entries limit 1`), quindi basta **una chiamata a buon
  fine ogni tanto** per azzerare il timer.
- `quiz-keepalive` **si auto-verifica**: legge la risposta e conferma nel log
  `mode=supabase` + `db=alive`. Se il servizio fosse in `mode=memory` (Supabase
  non configurato) o la chiamata fallisse, il job va in rosso e manda la mail.

### Fasce orarie e ora legale (Order)

Order deve stare sveglio **09:00–00:59 Europe/Rome** e dormire il resto della
notte (per risparmiare le **750 ore/mese condivise** di Render Free).

GitHub Actions gira **sempre in UTC** e **non gestisce l'ora legale**, quindi
un cron fisso "sballerebbe" di un'ora tra estate e inverno. Perciò i workflow
di Order girano su una fascia UTC ampia (`7-23`) ma **dentro il job** leggono
l'ora vera di `Europe/Rome` e agiscono solo se sono nella fascia 09:00–00:59.
Risultato: **corretto tutto l'anno, senza ritocchi stagionali**.

## Avvisi email

Gli avvisi usano le **notifiche native di GitHub**: quando un job fallisce,
GitHub invia un'email al proprietario del repo (indirizzo in *Settings →
Notifications / Emails*). Nessun segreto, nessuna configurazione extra.

> Suggerimento: le mail di `notifications@github.com` possono finire in
> **Promozioni/Spam**. Segnala "non è spam" o crea un filtro per portarle in
> Posta in arrivo.

## Verifica rapida

1. Tab **Actions** → scegli un workflow → **"Run workflow"**.
2. Apri il run → job → espandi lo step e leggi il log:
   - Order: `HTTP 200 in ...s`
   - Quiz: `OK: Supabase interrogato davvero (mode=supabase, db=alive)`
3. Per provare l'avviso email: basta che un servizio sia davvero giù (o si può
   fare un test temporaneo con un job che fallisce apposta).

## Note

- **cron-job.org è dismesso**: sia Order che Quiz sono qui.
- Il branch di default è `main`; i workflow girano da lì.
