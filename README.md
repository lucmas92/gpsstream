# GPS Stream

Una webapp semplice per il tracciamento GPS che ottiene la posizione corrente dell'utente utilizzando l'API di geolocalizzazione del browser e la invia a un server backend tramite una richiesta POST.

## Tecnologie Utilizzate

- **Vue 3**: Framework JavaScript per la costruzione dell'interfaccia utente.
- **Vite**: Strumento di build veloce per lo sviluppo frontend.
- **Vite Plugin PWA**: Per abilitare le funzionalità Progressive Web App (PWA).

## Installazione

1. Clona il repository:
   ```
   git clone <url-del-repository>
   cd gpsstream
   ```

2. Installa le dipendenze:
   ```
   npm install
   ```

## Avvio

Per avviare il server di sviluppo:
```
npm run dev
```

Per costruire per la produzione:
```
npm run build
```

Per visualizzare l'anteprima della build:
```
npm run preview
```

## Utilizzo

1. Apri la webapp nel browser.
2. Clicca sul pulsante "Ottieni posizione" per recuperare la tua posizione GPS corrente.
3. La posizione verrà visualizzata e inviata automaticamente al backend all'endpoint `/api/location`.

Nota: Assicurati che il browser supporti la geolocalizzazione e che l'utente conceda i permessi necessari.

## Licenza

[Inserisci la licenza se applicabile]
