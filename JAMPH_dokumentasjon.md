# JAMPH Ollama — Dokumentasjon

Denne dokumentasjonen beskriver hvordan du bruker og konfigurerer JAMPH Ollama-tjenesten, inkludert hvordan du laster inn egne modeller fra HuggingFace.

Modeller kan lages og lastes opp på huggingface med JAMPH MLTrainer. Du trenger derfor ikke kunnskap om hvordan.

---

## Innholdsfortegnelse

1. [Oversikt](#oversikt)
2. [Legg til din egen HuggingFace-modell i Docker-imaget](#legg-til-din-egen-huggingface-modell-i-docker-imaget)
3. [Last ned modell via API fra Kotlin](#last-ned-modell-via-api-fra-kotlin)
4. [Bruke modeller i applikasjonen](#bruke-modeller-i-applikasjonen)
---

## Oversikt

JAMPH Ollama kjører som en containerbasert LLM-tjeneste. Modeller kan enten:
- **Bakes inn** i Docker-imaget på byggetidspunktet (anbefalt for produksjon på NAIS)
- **Lastes ned dynamisk** via Ollama-APIet ved oppstart eller etter behov

---

## Legg til din egen HuggingFace-modell i Docker-imaget

For å bytte ut eller legge til en modell som bakes inn i imaget, rediger `model-downloader`-steget i `Dockerfile.build-from-source`.

### Forutsetninger

Modellen din på HuggingFace må være i **GGUF-format**. Hvis du har kvantisert modellen selv, last opp `.gguf`-filen til et HuggingFace-repo før du fortsetter.

### Steg 1 — Finn modell-URLen din

Format for HuggingFace-modeller i Ollama:

```
hf.co/<brukernavn>/<repo-navn>:<filnavn-uten-.gguf>
```

Eksempel:
```
hf.co/ditt-navn/min-sql-modell:Min-SQL-Modell-Q4_K_M
```

### Steg 2 — Oppdater Dockerfile

Finn `ollama pull`-linjen i `model-downloader`-steget og bytt ut modellnavnet:

```dockerfile
# Bytt ut denne linjen:
timeout 900 ollama pull qwen2.5-coder:7b && \

# Med din HuggingFace-modell:
timeout 900 ollama pull hf.co/ditt-navn/min-sql-modell:Min-SQL-Modell-Q4_K_M && \
```

Husk også å oppdatere referansen til modellnavnet der det brukes i applikasjonen (API-kall, konfigurasjon o.l.).

### Steg 3 — Bygg imaget på nytt

```bash
docker build -f Dockerfile.build-from-source -t jamph-ollama:secure .
```

---

## Last ned modell via API fra Kotlin

Ollama eksponerer et REST-API for å laste ned modeller programmatisk. Dette er nyttig dersom du ønsker å laste ned en modell uten å bygge imaget på nytt.

### Endepunkt

```
POST http://localhost:11434/api/pull
```

### Kotlin-eksempel med `ktor-client`

```kotlin
import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

@Serializable
data class PullRequest(
    val name: String,
    val stream: Boolean = false
)

suspend fun pullModel(modelName: String) {
    val client = HttpClient(CIO)

    val response: HttpResponse = client.post("http://localhost:11434/api/pull") {
        contentType(ContentType.Application.Json)
        setBody(Json.encodeToString(PullRequest(name = modelName)))
    }

    println("Status: ${response.status}")
    println("Respons: ${response.bodyAsText()}")

    client.close()
}

// Bruk:
// pullModel("hf.co/ditt-navn/min-sql-modell:Min-SQL-Modell-Q4_K_M")
```

> **Merk:** `stream: false` betyr at du får ett samlet JSON-svar når nedlastingen er ferdig. Utelater du det (eller setter `stream: true`), får du én JSON-linje per fremdriftsoppdatering — nyttig for å vise fremdrift i UI.

## Bruke modellen i applikasjonen

Etter at modellen er lastet ned (enten bakt inn eller hentet via API), kan du sende spørsmål til den via `/api/generate`-endepunktet.

### Endepunkt

```
POST http://localhost:11434/api/generate
```

### Kotlin-eksempel med `ktor-client`

```kotlin
import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

@Serializable
data class GenerateRequest(
    val model: String,
    val prompt: String,
    val stream: Boolean = false
)

@Serializable
data class GenerateResponse(
    val response: String
)

suspend fun generateSql(modelName: String, prompt: String): String {
    val client = HttpClient(CIO)

    val httpResponse: HttpResponse = client.post("http://localhost:11434/api/generate") {
        contentType(ContentType.Application.Json)
        setBody(Json.encodeToString(GenerateRequest(model = modelName, prompt = prompt)))
    }

    val body = httpResponse.bodyAsText()
    client.close()

    return Json.decodeFromString<GenerateResponse>(body).response
}

// Bruk:
// val sql = generateSql("hf.co/ditt-navn/min-sql-modell:Min-SQL-Modell-Q4_K_M", "Skriv SQL for å hente alle brukere opprettet siste 7 dager")
// println(sql)
```

> **Merk:** Modellnavnet må stemme eksakt med navnet som ble brukt ved nedlasting (f.eks. fra `ollama list` eller `/api/tags`).

---