# pucciotto-keepalive

"Sala di controllo" che tiene svegli i servizi **Render Free** durante la loro
fascia operativa, usando un **GitHub Action** schedulato.

## Perché esiste

I servizi su Render Free si addormentano dopo ~15 min di inattività e il
**risveglio a freddo dura ~50 secondi**. Il piano gratuito di **cron-job.org**
ha un **timeout massimo di 30s**: staccava la connessione *prima* che Render
finisse di avviarsi, quindi il servizio non si svegliava mai davvero (loop di
avvii annullati a metà).

La soluzione qui è un `curl` che **aspetta fino a ~150s** (`--max-time 150`):
tiene aperta la connessione oltre il cold start, così Render completa l'avvio e
il servizio resta caldo. GitHub Actions ha **minuti illimitati sui repo
pubblici**, quindi è gratis e sostenibile.

> Questo repo è **pubblico apposta** (minuti Actions illimitati) e **non
> contiene alcun segreto**: il workflow chiama solo URL pubblici.

## Cosa fa il workflow

File: [`.github/workflows/keepalive.yml`](.github/workflows/keepalive.yml)

- Gira ogni **5 minuti** nella fascia **07:00–23:59 UTC**.
- Sveglia **OrderPucciotto** → `https://orderpucciotto.onrender.com/ping`.
- Si può avviare a mano dal bottone **"Run workflow"** (evento
  `workflow_dispatch`).
- Se il `curl` fallisce, il job **non va in errore** (niente rumore).

### Fasce orarie e ora legale

GitHub Actions usa **sempre UTC** e **non gestisce l'ora legale**. La fascia
operativa di Order è ~**09:00–00:59 Europe/Rome**:

| Stagione | Offset | Fascia locale → UTC |
|----------|--------|----------------------|
| Estate (CEST) | UTC+2 | 09:00–00:59 = **07:00–22:59 UTC** |
| Inverno (CET) | UTC+1 | 09:00–00:59 = **08:00–23:59 UTC** |

Il cron `*/5 7-23 * * *` copre **tutta** la fascia in **entrambe** le stagioni,
con al massimo ~1h di margine ai bordi. Nelle ore centrali della notte
(00:00–07:00 UTC) Order resta spento, per rispettare le **750 ore/mese
condivise** di Render Free.

## Quiz (opzionale, oggi su cron-job.org)

Nel workflow c'è già uno step **commentato** per PuccioQuiz
(`/keep-db-alive`, serve solo a non far addormentare Supabase). Oggi Quiz è
ancora gestito da cron-job.org (doppio colpo 12:00 + 12:05) e va bene così.

Per consolidare tutto qui in futuro: scommentare quello step, aggiungere un
cron a mezzogiorno (es. `0 10 * * *` UTC = 12:00 CEST) e disattivare i job su
cron-job.org.

## Verifica

1. Tab **Actions** → workflow **keepalive** → "Run workflow" per un test manuale.
2. Nei log lo step di Order deve mostrare `HTTP 200`, anche partendo da servizio
   addormentato (il `--max-time` copre il cold start).

<!-- keepalive attivo -->
