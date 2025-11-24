# Sherlock MCP – Client Gids (MCP-server)

Deze MCP-server (“sherlock-mcp”) ontsluit Nederlandse RVO‑data rond subsidies, meldcodes/energie‑installaties en adresinformatie. De server is ontwikkeld door **Flowmatic** in samenwerking met **Techniek Nederland** en draait als FastMCP‑server achter een HTTP‑endpoint:

- MCP‑client URL: `https://sherlock-mcp-690462901472.europe-west4.run.app`  
- Authenticatie: momenteel **geen authenticatie** actief (iedere client met de URL kan verbinden)

Als MCP‑client zie je drie soorten capabilities:

- **Tools** – actieve functies (zoeken, ophalen, checks)
- **Resources** – statische/dynamische gegevens, o.a. JSON‑schema’s
- **Prompts** – herbruikbare prompt‑templates (voor je eigen LLM‑calls)

Onderstaand is een gids hoe je deze elementen als client gebruikt.

---

## 1. Verbinding maken met de Sherlock MCP‑server

De server draait als FastMCP‑HTTP‑server en is via MCP bereikbaar op de gegeven URL.

- In een MCP‑client (bv. Claude Desktop, VS Code MCP‑plugin, eigen integratie):
  - Configureer een **HTTP/streamable MCP‑server** met:
    - `url`: `https://sherlock-mcp-690462901472.europe-west4.run.app`
    - Geen extra headers of API keys (nu nog geen auth)
- De client zal bij connectie automatisch:
  - `initialize` uitvoeren
  - `tools/list`, `resources/list`, `prompts/list` ophalen  
    zodat alle mogelijkheden van de server bekend zijn.

---

## 2. Tools – de functionele kern

De server biedt een aantal gespecialiseerde tools aan. Belangrijkste tools:

### 2.1 `health_check`

Doel: status van de server en datadekking / versheid ophalen.

- Input (JSON): `HealthCheckInput`
  - `include_resources: bool` – of je per datasoort detail wilt.
- Output (JSON): `HealthCheckOutput`
  - `status`: `"ok"` of `"degraded"`
  - `server_name`, `timestamp`, `data_root`
  - `embedding_enabled`
  - `resources`: per resource recordcount, laatste refresh, stale‑flag (indien aangevraagd)

Gebruik in een client:

- Aanroepen bij start van een sessie om te checken of RVO‑data beschikbaar en actueel is.
- Eventueel tonen in UI (laatste update, welke datasets aanwezig zijn).

### 2.2 `address_lookup`

Doel: Nederlands adres vertalen naar gebouw-/monumentinformatie via Stella Spark Nexus WFS.

- Input (JSON): `AddressLookupInput`
  - `postal_code: "1012AB"` (string, zonder spatie)
  - `house_number: int`
  - `house_number_addition: str | null` (optioneel, bv. `"A"`)
- Output (JSON): `AddressLookupOutput`
  - `properties_list`: lijst met gebouw‑eigenschappen
  - `monument_list`: eventuele monumentinformatie
  - `cql_filter`: onderliggende WFS‑filter
  - `match_count`: aantal ruwe matches

Typische workflows:

- Je UI laat gebruiker een adres invoeren → je roept de tool aan → je voegt de eigenschappen/monumentstatus toe aan de context voor latere LLM‑analyse, of toon je direct in de app.

### 2.3 `search_rvo_subsidies`

Doel: semantische zoektool in de RVO‑subsidiecatalogus.

- Input (JSON): `SubsidySearchInput`
  - `query: str` – vrije tekst, bv. “warmtepomp hoekwoning 2025”
  - `statuses: list[str] | null` – filter op status (optioneel)
  - `limit: int` – max aantal resultaten
- Output (JSON): `SubsidySearchOutput`
  - `results`: lijst met subsidies (id, titel, url, scores, metadata)
  - `last_refresh_at`: laatste update van de dataset

Typische workflows:

- Bij persoonlijke subsidie‑adviezen: eerst `search_rvo_subsidies` draaien met de context van de offerte / woonhuis; resultaten als bronmateriaal opnemen in je LLM‑prompt of als keuzelijst tonen.

### 2.4 `search_rvo_meldcodes`

Doel: zoeken in meldcode‑/installatieregisters (isolatie, warmtepompen, zonneboilers, HR‑glas).

- Input (JSON): `MeldcodeSearchInput`
  - `installation_type: ResourceType` – bv. “heat_pumps” (exacte waarden in `ResourceType`)
  - `queries: list[str]` – één of meer zoektermen (productnaam, merk, type…)
  - `limit: int` – max matches per query
- Output (JSON): `MeldcodeSearchOutput`
  - Per query een lijst met matches; elke match bevat productmetadata en timestamps.

Typische workflows:

- Offerte analyseren → per installatie een query samenstellen (“Nefit EnviLine 6 kW”) → `search_rvo_meldcodes` → meldcodes + subsidiebedragen teruggeven.

### 2.5 ChatGPT‑compatibele tools: `search` en `fetch`

Doel: aansluiten op ChatGPT’s generieke “search & fetch”‑mechanisme.

- `search(query: str)`:
  - Zoekt over subsidies én meldcode‑resources.
  - Geeft terug: `{"results": [{"id": "...", "title": "...", "url": "..."}, ...]}`
  - `id` volgt patronen:
    - Subsidie: `subsidy|<subsidie_id>`
    - Meldcode: `meldcode|<resource_type>|<record_id>`
- `fetch(doc_id: str)`:
  - Verwacht een ID uit `search`.
  - Retourneert één volledig document:
    - `id`, `title`, `text` (samengestelde content), `url`, `metadata`

Typische workflows:

- In ChatGPT of een andere LLM‑host die generieke “search/fetch” kent, kun je deze server gewoon als zoekbron gebruiken zonder de RVO‑details te kennen.
- In een eigen client kun je `search` gebruiken voor snelle “top result”‑lists en `fetch` voor volledige inhoud en citaten.

#### Tools aanroepen in een client

- In programmeerbare clients (zoals de FastMCP Client) gebruik je `list_tools()` om tools te ontdekken en `call_tool()` om ze aan te roepen met JSON‑argumenten.
- In UI‑gebaseerde clients (zoals ChatGPT/Claude) zorg je dat de tools **aan** staan; het model mag ze dan zelfstandig oproepen als dat nuttig is, op basis van de toolbeschrijvingen.

---

## 3. Resources – schema’s en data voor de LLM

De server stelt diverse resources beschikbaar voor schema’s en ondersteunende data. Belangrijke URIs:

### 3.1 `schema://analyse-schema` (AnalysisSchema)

Doel: JSON‑schema voor gestructureerde **analyse** van input (offertes, facturen, etc.).

Bevat o.a.:

- `client_metadata` – bedrijfsnaam, klantnaam, adres, datum, enz.
- Verschillende bullet‑lijsten: `core_information_bullets`, `work_description_bullets`, `financial_overview_bullets`, `installations_specifications_bullets`, `omgevingsdata_bullets`, `timeline_and_execution_bullets`
- `missing_information` – maximaal 5 vervolgvraag‑items met label, hint, etc.

Gebruik in een client:

- Lees via `resources/read` het schema in.
- Gebruik dit schema om:
  - Een `response_format` te definiëren in je LLM‑call (bij JSON‑capable modellen).
  - Uitgebreide instructies in je prompt te genereren (“volg exact dit schema”).

### 3.2 `schema://report-schema` (ReportSchema / ReportEnvelope)

Doel: JSON‑schema voor de **eindrapportage** (subsidierapport).

Structuur:

- `ReportEnvelope` met:
  - `content` (`ReportContent`):
    - `huidige_offerte`
    - `installaties` (met installatieregelrijen en meldcodes)
    - `subsidie_inzichten` (landelijk/provinciaal/gemeentelijk)
    - `overige_inzichten`
    - `samenvatting_en_aanbevelingen` (topregelingen, hiaten/risico’s, vervolgstappen)
  - `gegenereerd_op` (datum)

Gebruik in een client:

- Als doel‑schema voor de “eind‑LLM‑call”: na analyse en RVO‑search laat je de LLM exact dit schema vullen.
- In UI kun je dit schema ook gebruiken om velden en secties in een rapport‑document of PDF dynamisch te bouwen.

### 3.3 `data://web-search/allowed-domains` (AllowedDomains)

Doel: JSON‑lijst van domeinen die zijn toegestaan voor websearch‑filtering (o.a. NL‑gemeenten en provincies).

Gebruik:

- Als je client ook een eigen websearch‑agent heeft, kun je deze lijst gebruiken om zoekresultaten te whitelisten (alleen RVO, gemeenten, provincies etc.).
- Je kunt de lijst tonen of cachen aan de client‑kant.

---

## 4. Prompts – centrale systeem‑prompt voor Sherlock

De server stelt ook herbruikbare prompt‑templates beschikbaar.

### 4.1 Prompt `sherlock-system`

- Type: server-side prompt template  
- Doel: geeft de **canonieke Sherlock system prompt** terug als een MCP‑`Message`.
- De tekst bevat o.a. verwachtingen, rol en stijlrichtlijnen voor de agent. De datum `{current_date}` wordt dynamisch ingevuld.
- Output:
  - Een `Message` met `role="assistant"` en uitgebreide tekst (system‑niveau instructie).

Gebruik in een client:

- Haal de prompt `sherlock-system` op via je MCP‑client en gebruik het resultaat als **system‑prompt** voor je LLM‑sessie.
- Combineer dit met de JSON‑schema resources:
  - System prompt = rol + gedrag  
  - Resource schema = gewenste outputstructuur  
  - Tools = extra capabilities.

---

## 5. Typische end‑to‑end workflows in een client

Hier een paar concrete scenario’s hoe je prompts, resources en tools samen inzet.

### 5.1 Health & capabilities‑check bij start

- Stap 1: `health_check` aanroepen met `{"include_resources": true}`
- Stap 2: Resultaat tonen (status, laatste refresh subsidies/meldcodes)
- Stap 3: Op basis hiervan beslissen of je bepaalde features (bv. meldcode‑suggesties) aan of uit zet.

### 5.2 Offerteanalyse → Analyse‑schema

- Stap 1: `sherlock-system` prompt ophalen en als system message instellen.
- Stap 2: `schema://analyse-schema` lezen.
- Stap 3: User uploadt offerte/factuur; jouw client stuurt deze tekst + schema‑instructies naar het LLM en vraagt om output die exact voldoet aan het schema.
- Stap 4: Gevalideerde JSON terugkrijgen → direct te gebruiken in front‑end of vervolgstap (bijv. automatisch velden vullen).

### 5.3 Installatie‑ en subsidieadviezen

- Stap 1: Uit analyse komt een lijst installaties/adres → gebruik `address_lookup` om context (monument, type gebouw) op te halen.
- Stap 2: Per installatie `search_rvo_meldcodes` voor mogelijke meldcodes; combineer dat met `search_rvo_subsidies` voor subsidies.
- Stap 3: `schema://report-schema` lezen.
- Stap 4: Een tweede LLM‑call uitvoeren die alle resultaten omzet naar één gestructureerd rapport, volledig conform `ReportEnvelope`.

### 5.4 ChatGPT‑stijl “search & fetch”

- Stap 1: In een LLM‑host die generieke tools ondersteunt, stel je de server beschikbaar.
- Stap 2: Het model roept `search(query)` aan als de gebruiker om “informatie over subsidie X” vraagt.
- Stap 3: De gebruiker of het model kiest een ID, waarna `fetch(doc_id)` wordt aangeroepen.
- Stap 4: De `text` en `metadata` uit `fetch` worden gebruikt voor onderbouwing/citaties.

---

## 6. Authenticatie en veiligheid

- Momenteel: **geen authenticatie** – alle clients met de URL hebben toegang.

Aan clientzijde:

- Ga ervan uit dat je met productieachtige data werkt (RVO‑datasets). Behandel resultaten alsof ze uit een trusted bron komen, maar blijf er kritisch mee omgaan (LLM‑interpretatie kan fouten introduceren).
- Als je deze MCP‑server in een publieke client integreert, overweeg eigen rate‑limiting / usage‑controle.

---

## 7. Samenvatting – hoe gebruik je deze MCP als client?

- Configureer de MCP‑client met de URL `https://sherlock-mcp-690462901472.europe-west4.run.app`.
- Gebruik **prompts**:
  - Haal `sherlock-system` op en gebruik het als system prompt voor je LLM.
- Gebruik **resources**:
  - Lees `schema://analyse-schema` en `schema://report-schema` om je LLM‑output strikt te structureren.
  - Optioneel: gebruik `data://web-search/allowed-domains` om websearch te filteren.
- Gebruik **tools**:
  - `health_check` voor status.
  - `address_lookup` voor adres → gebouw/monumentdata.
  - `search_rvo_subsidies` en `search_rvo_meldcodes` voor inhoudelijke RVO‑informatie.
  - `search` + `fetch` voor generieke “zoek & haal” integratie (ChatGPT‑compatibel).

---

## 8. FastMCP Client – Sherlock MCP gebruiken vanuit Python

Naast generieke MCP‑clients (Claude Desktop, ChatGPT, etc.) kun je ook rechtstreeks vanuit Python praten met de Sherlock MCP‑server via de **FastMCP Client** (`fastmcp.Client`). Dit geeft je een simpele, programmeerbare interface om tools, resources en prompts aan te roepen.

### 8.1 Basisvoorbeeld

Onderstaand voorbeeld laat zien hoe je verbinding maakt met de Sherlock MCP‑server, de beschikbare capabilities ophaalt en een health‑check draait:

```python
import asyncio
from fastmcp import Client

MCP_URL = "https://sherlock-mcp-690462901472.europe-west4.run.app"

async def main() -> None:
    client = Client(MCP_URL)

    async with client:
        # Controleren of de server bereikbaar is
        await client.ping()

        # Basis-capabilities ophalen
        tools = await client.list_tools()
        resources = await client.list_resources()
        prompts = await client.list_prompts()

        print("Tools:", [t.name for t in tools.tools])
        print("Resources:", [r.uri for r in resources.resources])
        print("Prompts:", [p.name for p in prompts.prompts])

        # Voorbeeld: health_check tool uitvoeren
        health = await client.call_tool("health_check", {"include_resources": True})
        print("Health status:", health.data)

asyncio.run(main())
```

Je ziet dat je in dezelfde sessie tools, resources en prompts kunt gebruiken – precies de drie bouwstenen van Sherlock MCP.

### 8.5 Tools, resources en prompts via de FastMCP Client

De FastMCP Client ontsluit dezelfde functionaliteit die je eerder in deze gids zag – maar dan met extra ergonomie rond tooling.

#### 8.5.1 Tools ontdekken

Gebruik `list_tools()` om alle tools van Sherlock MCP op te halen:

```python
from fastmcp import Client

async with Client(MCP_URL) as client:
    tools = await client.list_tools()
    for tool in tools.tools:
        print("Tool:", tool.name)
        print("Beschrijving:", tool.description)
        if tool.inputSchema:
            print("Parameterschema:", tool.inputSchema)

```

Je kunt op tags filteren, bijvoorbeeld om alleen “analyse”‑tools te tonen:

```python
async with Client(MCP_URL) as client:
    tools = await client.list_tools()

    analyse_tools = [
        t for t in tools.tools
        if getattr(t, "meta", None)
        and "_fastmcp" in t.meta
        and "analysis" in t.meta["_fastmcp"].get("tags", [])
    ]

    print("Analyse-tools:", [t.name for t in analyse_tools])
```

> Opmerking: het `meta._fastmcp`‑blok is een FastMCP‑conventie; het kan per serverconfiguratie aan/uit staan.

#### 8.5.2 Tools uitvoeren

Een tool roep je aan met `call_tool(name, arguments=...)`:

```python
from fastmcp import Client

async with Client(MCP_URL) as client:
    result = await client.call_tool(
        "search_rvo_subsidies",
        {"query": "warmtepomp hoekwoning 2025", "limit": 5},
    )
    print(result.data)
```

Je kunt ook geavanceerde opties meegeven, zoals een `timeout` of een specifieke `progress_handler`:

```python
async def my_progress_handler(progress: float, total: float | None, message: str | None) -> None:
    print(f"{progress}/{total} - {message}")

async with Client(MCP_URL) as client:
    result = await client.call_tool(
        "search_rvo_meldcodes",
        {"installation_type": "heat_pumps", "queries": ["Nefit EnviLine 6 kW"], "limit": 3},
        timeout=5.0,
        progress_handler=my_progress_handler,
    )
```

Daarnaast kun je `meta` meesturen voor tracing of client‑informatie:

```python
async with Client(MCP_URL) as client:
    result = await client.call_tool(
        "health_check",
        {"include_resources": True},
        meta={"trace_id": "abc-123", "client": "sherlock-dashboard"},
    )
```

#### 8.5.3 Resultaten lezen (CallToolResult)

`call_tool()` retourneert een `CallToolResult` met drie belangrijke vlakken:

- `result.data` – **FastMCP‑exclusief**: volledig gehydrateerde Python‑objecten op basis van het output‑schema (incl. datetimes, enums, eigen Pydantic‑modellen, etc.).
- `result.structured_content` – de ruwe gestructureerde JSON (standaard MCP).
- `result.content` – lijst met standaard MCP content‑blocks (tekst, etc.).

Voor de Sherlock‑tools kun je in de praktijk bijna altijd `result.data` gebruiken:

```python
async with Client(MCP_URL) as client:
    result = await client.call_tool(
        "health_check",
        {"include_resources": True},
    )

    health = result.data  # reeds omgezet naar het HealthCheckOutput-model
    print("Status:", health.status)
    print("Server:", health.server_name)
    print("Timestamp:", health.timestamp)
```

#### 8.5.4 Foutafhandeling bij tools

Standaard gooit `call_tool()` een `ToolError` als de tool faalt:

```python
from fastmcp import Client
from fastmcp.exceptions import ToolError

async with Client(MCP_URL) as client:
    try:
        result = await client.call_tool("potentially_failing_tool", {"param": "value"})
        print("OK:", result.data)
    except ToolError as exc:
        print("Tool mislukt:", exc)
```

#### 8.5.5 Resources (schema’s en data) via de client

Resources zijn data‑bronnen die Sherlock MCP exposeert. Voor Sherlock zijn dit met name:

- `schema://analyse-schema` – JSON‑schema voor de analysestructuur.
- `schema://report-schema` – JSON‑schema voor de rapportstructuur.
- `data://web-search/allowed-domains` – JSON met toegestane domeinen.

Je ontdekt resources met `list_resources()`:

```python
async with Client(MCP_URL) as client:
    result = await client.list_resources()
    for resource in result.resources:
        print("URI:", resource.uri)
        print("Naam:", resource.name)
        print("Beschrijving:", resource.description)
        print("MIME-type:", resource.mimeType)

```

**Resource‑inhoud lezen**

Met `read_resource(uri)` lees je de inhoud van een resource:

```python
import json

async with Client(MCP_URL) as client:
    contents = await client.read_resource("schema://analyse-schema")
    for item in contents:
        text = getattr(item, "text", None)
        if text is not None:
            print("MIME-type:", item.mimeType)
            schema = json.loads(text)
            print("Top-level keys:", schema.keys())
```

Voor Sherlock‑resources is de inhoud tekstueel JSON (`mimeType="application/json"`), dus je kunt die direct parsen. Voor binaire content (bijv. afbeeldingen) kun je `item.blob` gebruiken en zelf naar disk schrijven.

**Resource templates**

FastMCP ondersteunt ook “resource templates” (URI‑patronen met parameters) via `list_resource_templates()` en `read_resource()` met een ingevulde template‑URI. Sherlock definieert op dit moment geen templates, maar een generieke client kan hier toch mee omgaan:

```python
async with Client(MCP_URL) as client:
    templates = await client.list_resource_templates()
    print("Templates:", [t.uriTemplate for t in templates.templates])
```

In een multi‑server‑client worden URIs doorgaans geprefixt met de servernaam (conventie per client), bijvoorbeeld `sherlock:schema://analyse-schema` – controleer de FastMCP‑clientdocumentatie voor het exacte patroon.

#### 8.5.6 Prompts via de client

Prompts zijn herbruikbare prompt‑templates die door de server worden aangeboden. Sherlock heeft in ieder geval de `sherlock-system` prompt (centrale systeem‑prompt), maar de client‑API werkt generiek voor alle prompts.

**Prompts ontdekken**

```python
async with Client(MCP_URL) as client:
    result = await client.list_prompts()
    for prompt in result.prompts:
        print("Prompt:", prompt.name)
        print("Beschrijving:", prompt.description)
        if prompt.arguments:
            print("Argumenten:", [arg.name for arg in prompt.arguments])

```

**Prompts gebruiken in een LLM‑flow**

Met `get_prompt(name, arguments)` vraag je de server om de prompt te renderen naar een lijst MCP‑berichten:

```python
async with Client(MCP_URL) as client:
    # Sherlock system prompt ophalen
    result = await client.get_prompt("sherlock-system", {})

    for message in result.messages:
        print("Rol:", message.role)
        print("Content:", message.content)
```

De geretourneerde `messages` kun je direct als system/assistant/user‑berichten doorgeven aan je LLM‑client (OpenAI, Anthropic, etc.).

**Prompts met argumenten en automatische serialisatie**

Als een prompt argumenten accepteert, geef je die mee als dict. FastMCP serializeert complexe waarden automatisch naar JSON‑strings volgens de MCP‑specificatie, zodat de server ze weer als getypeerde objecten kan inlezen:

```python
from dataclasses import dataclass

@dataclass
class UserContext:
    name: str
    segment: str

async with Client(MCP_URL) as client:
    result = await client.get_prompt(
        "analyse_offerte",
        {
            "user": UserContext(name="Alice", segment="zakelijk"),
            "config": {"include_subsidies": True, "language": "nl"},
            "title": "Analyse van offerte 123",
        },
    )

    for msg in result.messages:
        print(msg.role, ":", msg.content)
```
