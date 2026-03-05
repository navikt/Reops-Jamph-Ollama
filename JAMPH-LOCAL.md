# Kjøre Jamph Ollama lokalt

Denne guiden beskriver hvordan teamet kan kjøre den samme Ollama-versjonen lokalt som kjører i produksjon på NAIS — uten å installere noe utover Docker.

## Bygg imaget


```bash
docker build -f Dockerfile.build-from-source -t reops-ollama:secure .
```

## Kjør lokalt

```bash
docker run -p 11434:11434 reops-ollama:secure
```
Ollama er klar når du ser `Ollama is running` i loggen (tar typisk 10–30 sekunder etter oppstart).

## Verifiser at det virker

```bash
# Sjekk at tjenesten svarer
curl http://localhost:11434/api/version

# Se hvilke modeller som er lastet inn
curl http://localhost:11434/api/tags
```

## Bruk med API

Send et spørsmål til `qwen2.5-coder:7b` (samme modell som i prod):

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:7b",
  "prompt": "Write a SQL query that selects all users created in the last 7 days.",
  "stream": false
}'
```

## Stopp containeren

```bash
# Finn container-ID
docker ps

# Stopp
docker stop <container-id>
```

## Feilsøking

| Problem | Løsning |
|---|---|
| Port 11434 er opptatt | Finn og stopp prosessen som bruker porten: `netstat -ano \| findstr :11434`, deretter `taskkill /PID <pid> /F` |
| Bygget feiler på `cmake` | Øk Docker Desktop Memory til minst 6 GB (Settings → Resources) |
| Modellen svarer ikke | Vent litt lenger — `qwen2.5-coder:7b` tar noen sekunder å laste inn i minnet |
| `docker: image not found` | Du må bygge imaget først (se steg over) |

## Referanse

- Ollama-versjon i dette imaget: `v0.15.2`
- Modell: `qwen2.5-coder:7b`
- Bygd med Go `1.26rc3` (patcher CVE-2025-22871, CVE-2025-47907, CVE-2025-61723)
- Prod-miljø: `https://jamph-ollama.intern.dev.nav.no`
