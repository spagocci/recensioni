# Guida rapida: copiare il codice generato e pubblicarlo su GitHub

Questa guida spiega due operazioni comuni:

1. **Salvare in locale una copia del codice generato dal bot** (ad esempio `bot.js`).
2. **Caricare le modifiche su GitHub** tramite commit e push oppure aprendo una Pull Request.

Le istruzioni assumono che tu stia lavorando da riga di comando in questa cartella del progetto (`recensioni`).

---

## 1. Copiare il codice creato dal bot in un file di testo

Se il bot ha aggiornato un file (es. `bot.js`) e vuoi ottenere una copia di quanto contenuto:

```bash
# Mostra il contenuto a video
cat bot.js

# Copia il file in un percorso diverso
cp bot.js /percorso/destinazione/bot.js

# Oppure esporta in un file di testo personalizzato
cp bot.js bot_vetrina.txt
```

> Suggerimento: se lavori da un’interfaccia grafica, puoi aprire il file con un editor e copiare/incollare il contenuto.

---

## 2. Preparare e inviare modifiche su GitHub

### 2.1 Configurazione iniziale (solo la prima volta)

```bash
# Configura nome e email usati per i commit
git config user.name "Il Tuo Nome"
git config user.email "tua@email.it"
```

Assicurati di avere i permessi di scrittura sul repository remoto (SSH o HTTPS con token).

### 2.2 Creare il commit con le modifiche

```bash
# Controlla quali file sono cambiati
git status

# Aggiungi i file modificati all'area di staging
git add bot.js bot_vetrina.txt GUIDA_GITHUB.md

# Crea un commit con un messaggio descrittivo
git commit -m "Spiega come copiare il codice e caricarlo su GitHub"
```

Se vuoi includere tutti i file modificati puoi usare `git add .`.

### 2.3 Caricare le modifiche su GitHub

#### Opzione A: Push diretto sul branch principale

```bash
# Invia il commit al ramo remoto (es. main)
git push origin main
```

Il commit comparirà subito su GitHub.

#### Opzione B: Creare una Pull Request

1. Crea un nuovo ramo per il tuo lavoro:
   ```bash
   git checkout -b nome-del-tuo-ramo
   ```
2. Aggiungi e committa le modifiche come sopra.
3. Invia il ramo su GitHub:
   ```bash
   git push origin nome-del-tuo-ramo
   ```
4. Vai su GitHub e apri una Pull Request dal ramo appena caricato verso `main`.

---

### 2.4 Aggiornare il repository locale

Per sincronizzarti con eventuali modifiche fatte da altri:

```bash
git pull origin main
```

---

## 3. Risoluzione dei problemi comuni

| Problema | Possibile soluzione |
| --- | --- |
| Git chiede le credenziali | Genera un [token personale su GitHub](https://github.com/settings/tokens) e usalo al posto della password oppure configura una chiave SSH. |
| Conflitti durante il `git pull` | Apri i file indicati, risolvi manualmente i conflitti, poi esegui `git add` sui file risolti e `git commit`. |
| Non vedi il file su GitHub | Controlla di aver fatto `git push` sul ramo corretto e di avere i permessi necessari. |

---

## 4. Ulteriori risorse utili

- [Documentazione ufficiale Git](https://git-scm.com/doc)
- [Documentazione Pull Requests GitHub](https://docs.github.com/pull-requests)
- [Guida GitHub per principianti (in italiano)](https://docs.github.com/it/get-started)

Conserva questo file come riferimento rapido per le prossime modifiche.
