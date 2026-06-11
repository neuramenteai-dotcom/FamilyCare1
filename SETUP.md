# FamilyCare — Guida attivazione servizi reali

Il sito funziona già in **modalità demo** (account in localStorage, email simulate a schermo, verifica identità simulata).
Per attivare i servizi reali bastano 3 account gratuiti. Le chiavi vanno incollate nel blocco `CONFIG` in fondo a `index.html` (cerca `var CONFIG=`).

---

## 1. Verifica identità reale — Didit (GRATIS, 500 verifiche/mese per sempre)

Didit verifica documento + selfie con liveness e face match reali. Piano gratuito: 500 verifiche/mese, senza carta di credito.

1. Registrati su **https://business.didit.me**
2. Crea un **Workflow** di tipo KYC (ID Verification + Liveness + Face Match sono inclusi nel piano free)
3. Vai su **Unilinks** (o "Verification Links") e crea lo **Unilink** del workflow: è un link riutilizzabile, senza backend
4. Copia il link (es. `https://verify.didit.me/...`) e incollalo in `CONFIG.DIDIT_UNILINK`

Da quel momento le collaboratrici che si iscrivono faranno la **verifica vera** sulla pagina Didit. Gli esiti li vedi nel Business Console Didit.

> Conferma automatica dell'esito sul sito: serve un webhook (Didit → un piccolo endpoint serverless, es. Supabase Edge Function o Cloudflare Worker). Senza webhook l'utente preme "Ho completato la verifica" e tu controlli l'esito nel console Didit prima di pubblicare il profilo.

---

## 2. Account reali + email di conferma reali — Supabase (GRATIS)

Supabase gestisce registrazione, login e **invia davvero l'email di conferma** con link di attivazione.

1. Crea un progetto su **https://supabase.com** (piano Free)
2. **Settings → API**: copia `Project URL` → `CONFIG.SUPABASE_URL` e la chiave `anon public` → `CONFIG.SUPABASE_ANON_KEY`
3. **Authentication → URL Configuration**: imposta come *Site URL*:
   `https://neuramenteai-dotcom.github.io/FamilyCare1/`
4. **SQL Editor** → esegui questo script per la tabella dei profili:

```sql
create table profili_collaboratrici (
  id bigint primary key generated always as identity,
  created_at timestamptz default now(),
  nome text not null,
  email text,
  cat text default 'assistente',
  zona text default 'Roma',
  descrizione text,
  skills jsonb default '[]',
  kyc boolean default false
);

alter table profili_collaboratrici enable row level security;

-- chiunque può iscriversi (insert), solo gli utenti loggati leggono i profili
create policy "iscrizione pubblica" on profili_collaboratrici
  for insert with check (true);
create policy "lettura per utenti registrati" on profili_collaboratrici
  for select using (auth.role() = 'authenticated');
```

Con Supabase attivo:
- la registrazione invia una **email vera** con link di conferma (mittente Supabase; max ~4/ora sul piano free — per volumi reali collega un SMTP tuo in Authentication → Emails)
- login/logout veri, la sessione sopravvive al refresh
- i profili delle collaboratrici si salvano nel **database** e sono visibili a tutte le famiglie registrate, da qualsiasi dispositivo

---

## 3. Email di notifica dal sito — EmailJS (GRATIS, 200 email/mese)

Per le email "di cortesia" (conferma richiesta famiglia, benvenuto collaboratrice).

1. Account su **https://www.emailjs.com** → collega un servizio email (es. Gmail)
2. Crea un **Template** con queste variabili: `{{to_email}}` (destinatario), `{{subject}}`, `{{message}}`
   - nel campo "To email" del template metti `{{to_email}}`
3. Copia in `CONFIG`: **Public Key** (Account → General), **Service ID**, **Template ID**

---

## Dove incollare le chiavi

In `index.html`, in fondo al file, trova:

```js
var CONFIG={
  SUPABASE_URL:'',          // ← qui
  SUPABASE_ANON_KEY:'',     // ← qui
  DIDIT_UNILINK:'',         // ← qui
  EMAILJS_PUBLIC_KEY:'',    // ← qui
  EMAILJS_SERVICE_ID:'',    // ← qui
  EMAILJS_TEMPLATE_ID:''    // ← qui
};
```

Ogni servizio si attiva da solo appena la sua chiave non è vuota; quelli senza chiave restano in demo.
Apri la console del browser (F12) per vedere lo stato: `[FamilyCare] Supabase:ATTIVO · Didit:demo · ...`

> ⚠️ La chiave `anon public` di Supabase e le chiavi EmailJS sono pensate per stare nel front-end: nessun segreto viene esposto. Non incollare MAI nel sito chiavi segrete (Didit API key, Supabase service_role).

---

## Limiti noti senza backend

| Funzione | Stato | Per averla al 100% |
|---|---|---|
| Conferma automatica esito Didit | manuale (console Didit) | webhook + serverless function |
| Email transazionali illimitate | 200/mese (EmailJS free) | SMTP proprio su Supabase |
| Password dimenticata | non gestita | si attiva con Supabase (`resetPasswordForEmail`) |
