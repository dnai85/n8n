# Paperless-NGX → n8n → Apple Reminders Workflow

## Übersicht
Dieser Workflow überwacht Paperless-NGX Dokumente mit dem Tag "todo" und erstellt automatisch Apple Reminders mit den Dokumentdetails.

## Komponenten
1. **Paperless-NGX**: Dokumentenverwaltung mit Tagging
2. **n8n**: Workflow-Automatisierung
3. **Apple Shortcuts**: Webhook-Empfang und Reminder-Erstellung

---

## 1. Paperless-NGX Konfiguration

### API-Token erstellen
1. In Paperless-NGX zu Settings → API Tokens
2. Neuen Token für n8n erstellen
3. Token sicher speichern

### Webhook URL notieren
- Paperless-NGX URL: `https://your-paperless-instance.com`
- API Endpoint: `/api/documents/`

---

## 2. n8n Workflow Setup

### Workflow-Knoten

#### Knoten 1: Trigger (Cron)
```json
{
  "parameters": {
    "rule": {
      "interval": [
        {
          "field": "cronExpression",
          "expression": "*/5 * * * *"
        }
      ]
    }
  },
  "name": "Every 5 minutes",
  "type": "n8n-nodes-base.cron"
}
```

#### Knoten 2: HTTP Request - Paperless API
```json
{
  "parameters": {
    "url": "https://your-paperless-instance.com/api/documents/",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "options": {
      "queryParameters": {
        "parameters": [
          {
            "name": "tags__name",
            "value": "todo"
          },
          {
            "name": "ordering",
            "value": "-created"
          }
        ]
      }
    }
  },
  "name": "Get Todo Documents",
  "type": "n8n-nodes-base.httpRequest"
}
```

#### Knoten 3: Function - Process Documents
```javascript
// Verarbeitung der Paperless-NGX Antwort
const documents = $input.first().json.results;
const processedItems = [];

for (const doc of documents) {
  // Prüfen ob bereits verarbeitet (hier könntest du eine Datenbank oder Memory-Store nutzen)
  processedItems.push({
    id: doc.id,
    title: doc.title,
    created: doc.created,
    content: doc.content,
    tags: doc.tags,
    download_url: `https://your-paperless-instance.com/api/documents/${doc.id}/download/`,
    web_url: `https://your-paperless-instance.com/documents/${doc.id}/`
  });
}

return processedItems.map(item => ({ json: item }));
```

#### Knoten 4: HTTP Request - Webhook zu Apple Shortcuts
```json
{
  "parameters": {
    "url": "{{ $('Apple Shortcuts Webhook').first().json.webhook_url }}",
    "sendBody": true,
    "bodyParameters": {
      "parameters": [
        {
          "name": "document_id",
          "value": "={{ $json.id }}"
        },
        {
          "name": "title",
          "value": "={{ $json.title }}"
        },
        {
          "name": "created",
          "value": "={{ $json.created }}"
        },
        {
          "name": "web_url",
          "value": "={{ $json.web_url }}"
        },
        {
          "name": "download_url",
          "value": "={{ $json.download_url }}"
        }
      ]
    }
  },
  "name": "Send to Apple Shortcuts",
  "type": "n8n-nodes-base.httpRequest"
}
```

---

## 3. Apple Shortcuts Konfiguration

### Schritt 1: Webhook Shortcut erstellen

1. **Shortcuts App öffnen**
2. **Neuer Shortcut**: "Paperless Todo Reminder"
3. **Aktionen hinzufügen**:

#### Aktion 1: Get Contents of URL
```
URL: Wird automatisch vom Webhook übertragen
Method: POST
Request Body: JSON
```

#### Aktion 2: Get Value for Key
```
Dictionary: Contents of URL (aus vorheriger Aktion)
Key: title
```

#### Aktion 3: Get Value for Key
```
Dictionary: Contents of URL
Key: web_url
```

#### Aktion 4: Get Value for Key
```
Dictionary: Contents of URL
Key: created
```

#### Aktion 5: Format Date
```
Date: Get Value for "created"
Format: Medium Style
```

#### Aktion 6: Text
```
Text: "📄 Paperless Dokument: [Value from step 2]

Erstellt: [Formatted Date from step 5]

Link: [Value from step 3]

#paperless #todo"
```

#### Aktion 7: Add New Reminder
```
List: Todo (oder deine gewünschte Liste)
Title: [Value from step 2] (Dokumenttitel)
Notes: [Text from step 6]
Alert: Optional - z.B. in 1 Stunde
```

### Schritt 2: Webhook URL generieren

1. **Shortcut bearbeiten**
2. **Settings** (Zahnrad-Symbol)
3. **Use with Siri** aktivieren
4. **Add to Siri**
5. **Privacy**: "Anyone" (für Webhook-Zugriff)
6. **Shortcut URL kopieren**

Die URL hat das Format:
```
https://www.icloud.com/shortcuts/[shortcut-id]
```

---

## 4. Erweiterte Konfiguration

### n8n Memory/Database für verarbeitete Dokumente

#### Redis/Memory Store
```javascript
// In Function Node - Duplikat-Prüfung
const processedDocs = await $('Redis').getItem('processed_paperless_docs') || [];
const newDocs = documents.filter(doc => !processedDocs.includes(doc.id));

// Nach erfolgreichem Versand
await $('Redis').setItem('processed_paperless_docs', 
  [...processedDocs, ...newDocs.map(d => d.id)]
);
```

### Fehlerbehandlung in n8n

#### Error Trigger Node
```json
{
  "parameters": {
    "errorWorkflows": [
      "Error Notification Workflow"
    ]
  },
  "name": "Error Handler",
  "type": "n8n-nodes-base.errorTrigger"
}
```

### Apple Shortcuts Erweitert

#### Conditional Logic für verschiedene Tags
```
If (Contains "urgent" in tags):
  - Set Alert: In 15 minutes
  - Add to "Urgent" list
Else:
  - Set Alert: Tomorrow 9 AM
  - Add to "Todo" list
```

---

## 5. Testing & Deployment

### Test-Schritte

1. **Paperless Test**:
   ```bash
   curl -H "Authorization: Token your-token" \
        "https://your-paperless-instance.com/api/documents/?tags__name=todo"
   ```

2. **n8n Test**:
   - Manual Trigger ausführen
   - Logs überprüfen

3. **Apple Shortcuts Test**:
   - Webhook URL direkt aufrufen:
   ```bash
   curl -X POST "https://www.icloud.com/shortcuts/[shortcut-id]" \
        -H "Content-Type: application/json" \
        -d '{"title":"Test Doc","web_url":"https://example.com"}'
   ```

### Deployment Checklist

- [ ] Paperless-NGX API Token konfiguriert
- [ ] n8n Workflow aktiviert
- [ ] Apple Shortcuts Webhook URL in n8n eingetragen
- [ ] Test-Dokument mit "todo" Tag erstellt
- [ ] Reminder in Apple Reminders App überprüft

---

## 6. Erweiterte Features

### Multi-Tag Support
```javascript
// In n8n Function Node
const tagActions = {
  'urgent': { alert: '15 minutes', list: 'Urgent' },
  'bills': { alert: 'due_date', list: 'Bills' },
  'todo': { alert: 'tomorrow 9am', list: 'Todo' }
};
```

### Rich Notifications
```
Shortcut Action: Show Notification
Title: "Neues Paperless Todo"
Body: "[Document Title] wurde hinzugefügt"
```

### Document Preview
```
Shortcut Action: Get Contents of URL
URL: [download_url from webhook]
-> Quick Look (für PDF Preview)
```

---

## Troubleshooting

### Häufige Probleme

1. **Webhook funktioniert nicht**:
   - Shortcuts Privacy auf "Anyone" setzen
   - iCloud Sync überprüfen

2. **n8n findet keine Dokumente**:
   - API Token überprüfen
   - Paperless URL und Pfad verifizieren

3. **Duplicate Reminders**:
   - Memory/Database für verarbeitete IDs implementieren
   - Timestamp-basierte Filterung

4. **Apple Shortcuts Fehler**:
   - JSON Format der Webhook-Daten überprüfen
   - Shortcuts App Berechtigungen für Reminders

### Debug Commands

```bash
# Paperless API Test
curl -H "Authorization: Token YOUR_TOKEN" \
     "https://your-paperless.com/api/documents/?tags__name=todo&limit=1"

# n8n Webhook Test
curl -X POST "http://your-n8n:5678/webhook/test" \
     -H "Content-Type: application/json" \
     -d '{"test": "data"}'
```