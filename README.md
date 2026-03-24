# Karmasutra

## Progetto Bot Quest Wolvesville + Telegram

Questo documento definisce la progettazione iniziale del bot richiesto dal cliente, con focus su gestione missioni/quest clan Wolvesville, annunci su Telegram e Wolvesville, e coordinamento voti/donazioni.

---

## 1) Requisiti funzionali

### 1.1 Obiettivi principali

Il bot deve:

1. Rilevare nuove quest disponibili nel clan Wolvesville.
2. Annunciare le nuove quest sia su Telegram sia su Wolvesville (annuncio/chat clan).
3. Invitare i membri a votare la quest preferita oppure shuffle.
4. Leggere i voti effettuati su Wolvesville.
5. Ricostruire elenco votanti (e, se collegati, mappare i votanti verso utenti Telegram).
6. Inviare messaggi di coordinamento donazioni (500 monete o 200 gemme in base alla missione).
7. Gestire scelta quest in modalità:
   - sicura (conferma admin)
   - automatica assistita (auto-setup partecipanti + conferma admin)
8. Gestire richiesta shuffle con soglia voti configurabile e conferma admin.
9. Monitorare quest attiva (participants/xp/tier/stato).
10. Annunciare completamento quest e riaprire ciclo di voto.

### 1.2 Requisiti di frequenza

- Il flusso non è “una quest a settimana”, ma tipicamente “una quest al giorno” (o comunque più frequente).
- Il bot deve quindi operare su ciclo continuo con polling periodico.
- Fascia del lunedì 06:00–08:00 UTC+locale clan da trattare come finestra ad alta priorità, senza vincolare la logica solo a tale fascia.

### 1.3 Requisiti canali di comunicazione

- **Telegram**: canale principale per mention, comandi admin, report e reminder.
- **Wolvesville clan**: canale ufficiale parallelo per annuncio/chat nel clan.

### 1.4 Requisiti di sicurezza operativa

Le azioni che consumano risorse clan (claim/shuffle/altre azioni costo) devono essere protette da:

- ruolo admin,
- conferma esplicita,
- audit log interno.

### 1.5 Requisiti di tracciabilità

Il bot deve salvare:

- eventi quest rilevati,
- voti snapshot,
- annunci inviati,
- mapping WW↔TG,
- azioni admin,
- stato ciclo corrente.

---

## 2) Architettura tecnica

### 2.1 Architettura logica

Componenti:

1. **Scheduler/Poller**
   - esegue polling API Wolvesville a intervallo configurabile (es. 3–5 minuti).
2. **Wolvesville API Client**
   - wrapper REST con autenticazione bot.
3. **Quest Engine (orchestratore stato)**
   - confronta stato precedente e stato nuovo, produce eventi di dominio.
4. **Notifier Telegram**
   - invio messaggi, mention, comandi admin.
5. **Notifier Wolvesville**
   - invio chat/announcement clan.
6. **Identity Linker WW↔TG**
   - collega player wolvesville a utente telegram.
7. **Persistence Layer (SQLite)**
   - storage snapshot e stato.
8. **Admin Action Guard**
   - policy per azioni sensibili con conferma.

### 2.2 Deployment target (PythonAnywhere free)

- Runtime Python 3.x.
- Scheduled task periodica (script entrypoint).
- SQLite locale come DB iniziale.
- Nessun requisito websocket persistente nella v1.

### 2.3 Moduli Python proposti

- `config.py`: variabili ambiente/config runtime.
- `wolvesville_client.py`: endpoint clan/quest/voti/annunci.
- `telegram_client.py`: invio messaggi Telegram e parsing comandi admin.
- `quest_engine.py`: logica di transizione stati.
- `identity_linker.py`: gestione mapping WW↔TG.
- `donation_coordinator.py`: messaggi di coordinamento donazioni.
- `admin_actions.py`: claim/shuffle/partecipanti con conferma.
- `repository.py`: CRUD su SQLite.
- `main.py`: entrypoint job periodico.

### 2.4 Eventi di dominio

- `NEW_QUESTS_AVAILABLE`
- `QUEST_VOTES_UPDATED`
- `SHUFFLE_THRESHOLD_REACHED`
- `QUEST_SELECTED_PENDING_CONFIRMATION`
- `QUEST_CLAIMED`
- `ACTIVE_QUEST_STARTED`
- `ACTIVE_QUEST_FINISHED`
- `DONATION_REMINDER_DUE`

L’orchestratore reagisce agli eventi e inoltra azioni ai moduli notifica e amministrazione.

---

## 3) Schema database (SQLite v1)

### 3.1 Tabelle principali

#### `clans`
- `id` TEXT PK (clanId wolvesville)
- `name` TEXT
- `is_active` INTEGER
- `created_at` DATETIME
- `updated_at` DATETIME

#### `users_link`
- `id` INTEGER PK AUTOINCREMENT
- `wolvesville_player_id` TEXT UNIQUE
- `wolvesville_username` TEXT
- `telegram_user_id` TEXT UNIQUE NULL
- `telegram_username` TEXT NULL
- `link_status` TEXT (`pending`,`verified`,`revoked`)
- `created_at` DATETIME
- `updated_at` DATETIME

#### `quest_catalog`
- `quest_id` TEXT PK
- `purchasable_with_gems` INTEGER
- `promo_image_url` TEXT
- `raw_payload` TEXT
- `created_at` DATETIME

#### `quest_cycles`
- `id` INTEGER PK AUTOINCREMENT
- `clan_id` TEXT FK -> clans.id
- `cycle_status` TEXT (`voting`,`pending_claim`,`active`,`completed`,`cancelled`)
- `available_hash` TEXT
- `selected_quest_id` TEXT NULL
- `is_shuffle_proposed` INTEGER
- `is_shuffle_executed` INTEGER
- `opened_at` DATETIME
- `closed_at` DATETIME NULL

#### `quest_votes_snapshot`
- `id` INTEGER PK AUTOINCREMENT
- `cycle_id` INTEGER FK -> quest_cycles.id
- `quest_id` TEXT
- `player_id` TEXT
- `vote_type` TEXT (`quest`,`shuffle`)
- `snapshot_at` DATETIME

#### `quest_participants`
- `id` INTEGER PK AUTOINCREMENT
- `cycle_id` INTEGER FK -> quest_cycles.id
- `player_id` TEXT
- `enabled` INTEGER
- `source` TEXT (`auto_from_votes`,`manual_admin`)
- `updated_at` DATETIME

#### `notifications_log`
- `id` INTEGER PK AUTOINCREMENT
- `cycle_id` INTEGER NULL FK -> quest_cycles.id
- `channel` TEXT (`telegram`,`wolvesville_chat`,`wolvesville_announcement`)
- `message_type` TEXT
- `content_hash` TEXT
- `sent_at` DATETIME
- `status` TEXT (`sent`,`failed`)
- `error_text` TEXT NULL

#### `admin_actions_log`
- `id` INTEGER PK AUTOINCREMENT
- `cycle_id` INTEGER NULL FK -> quest_cycles.id
- `admin_actor` TEXT
- `action_type` TEXT (`claim`,`shuffle`,`enable_participants`,`disable_participants`,`cancel`)
- `request_payload` TEXT
- `confirmed` INTEGER
- `executed` INTEGER
- `created_at` DATETIME
- `executed_at` DATETIME NULL

### 3.2 Vincoli principali

- Un solo ciclo `voting|pending_claim|active` aperto per clan.
- Deduplica notifiche con `content_hash` + finestra temporale.
- `users_link` deve consentire record non collegati Telegram (fallback username WW).

---

## 4) Flussi operativi dettagliati

### 4.1 Flusso A — rilevazione nuove quest

1. Poller richiede `quests/available`.
2. Engine calcola hash lista quest.
3. Se hash differente rispetto all’ultimo ciclo aperto:
   - apre nuovo ciclo (`voting`),
   - salva snapshot disponibilità,
   - invia annuncio TG + WW.
4. Messaggio standard:
   - “Sono uscite le nuove missioni/quest! Andate a votare. Se non vi piace nessuna, votate shuffle che ne ripeschiamo altre.”

### 4.2 Flusso B — raccolta voti e mapping utenti

1. Poller richiede `quests/votes`.
2. Engine salva snapshot voti quest + shuffle.
3. Per ogni `player_id`:
   - risolve username da cache membri clan,
   - tenta mapping a utente Telegram in `users_link`.
4. Produce report voti periodico solo se cambia lo stato (evita spam).

### 4.3 Flusso C — annuncio votanti + donazione

1. Su trigger (orario o raggiunta soglia voti), engine compone elenco votanti.
2. Telegram:
   - mention reali se mapping presente,
   - fallback `@username` testuale se assente.
3. Wolvesville:
   - elenco solo testuale username.
4. Messaggio donazione:
   - “Hanno votato: ... Donate ora 500 monete.”
   - variante 200 gemme se configurata per missione corrente.

### 4.4 Flusso D — scelta quest (safe vs auto-assistita)

#### Modalità safe (default)
1. Engine identifica quest più votata.
2. Crea proposta `pending_claim`.
3. Invia messaggio admin con richiesta conferma.
4. Solo dopo conferma esegue claim.

#### Modalità auto-assistita
1. Engine identifica quest vincente.
2. Esegue `disable all participateInQuests`.
3. Abilita `participateInQuests=true` solo per votanti quest selezionata.
4. Richiede conferma admin finale.
5. Esegue claim quest.

### 4.5 Flusso E — shuffle controllato

1. Engine conta `shuffleVotes`.
2. Se `shuffleVotes >= soglia`:
   - genera proposta shuffle,
   - notifica admin.
3. Solo su conferma admin esegue shuffle.
4. Dopo shuffle forza nuovo ciclo di rilevazione available quest.

### 4.6 Flusso F — monitor quest attiva

1. Poller richiede `quests/active`.
2. Se transizione `no active -> active`:
   - annuncio partenza quest (TG + WW).
3. Se transizione di stato/tier rilevante:
   - eventuale update (configurabile, anti-spam).
4. Se transizione `active -> none` o marcata completata:
   - annuncio completamento,
   - chiusura ciclo,
   - riapertura monitor per nuova votazione.

### 4.7 Flusso G — donazioni (coordinamento)

1. Bot non forza donazioni utente.
2. Bot invia reminder ai votanti.
3. Opzionale v2: confronto indicativo ledger/statistiche per report contributi.
4. In caso dati non conclusivi: output esplicito “stato donazione non verificabile in modo certo”.

---

## 5) Configurazione operativa consigliata (v1)

- Polling interval: 5 minuti.
- Cooldown notifiche stesso tipo: 20 minuti.
- Soglia shuffle default: 3 voti (configurabile).
- Modalità default: `safe`.
- Orario report riepilogo: 2 volte al giorno.

---

## 6) Roadmap implementazione

### Milestone 1 (MVP)
- Client API WW + notifier TG + notifier WW.
- Rilevazione available/votes/active.
- Annunci base e reminder donazione.
- Persistenza SQLite minima.

### Milestone 2
- Linking WW↔TG.
- Workflow admin conferma claim/shuffle.
- Modalità auto-assistita partecipanti.

### Milestone 3
- Report avanzati contributi.
- Dashboard operativa (opzionale).
- Policy anti-spam e osservabilità estesa.

