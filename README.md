# Sherlock MCP – Client Gids (MCP-server)

Deze MCP-server ("sherlock-mcp") ontsluit Nederlandse RVO‑data rond subsidies, meldcodes/energie‑installaties en adresinformatie. De server is ontwikkeld door **Flowmatic** in samenwerking met **Techniek Nederland** en draait als FastMCP‑server achter een HTTP‑endpoint:

- MCP‑client URL: `https://sherlock-mcp-351325125041.europe-west4.run.app/mcp`  
- Authenticatie: **API key vereist** – verzend als Bearer token: `Authorization: Bearer <YOUR_API_KEY>`

Als MCP‑client zie je drie soorten capabilities:

- **Tools** – functies (zoeken naar subsidies, meldcodes, adressen)
- **Resources** – schemas voor structured output, lijst van rijks/provincie/gemeentelijke domeinen.
- **Prompts** – sherlock system-prompt voor persona en instructies

Onderstaand is een gids hoe je deze elementen als client gebruikt.

---

## 1. Verbinding maken met de Sherlock MCP‑server

De server draait als FastMCP‑HTTP‑server en is via MCP bereikbaar op de gegeven URL.

- In een MCP‑client (bv. Claude Desktop, VS Code MCP‑plugin, eigen integratie):
  - Configureer een **MCP‑server** met:
    - `url`: `https://sherlock-mcp-351325125041.europe-west4.run.app/mcp`
    - `Authorization` header: `Bearer <YOUR_API_KEY>`
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
  - `include_resources: bool` – of je per datasoort detail wilt (default: `true`).
- Output (JSON): `HealthCheckOutput`
  - `status`: `"ok"` of `"degraded"`
  - `server_name`, `timestamp`
  - `storage_mode`, `storage_detail`
  - `embedding_enabled`
  - `resources`: per resource recordcount, laatste refresh, stale‑flag (indien aangevraagd)

Gebruik in een client:

- Aanroepen bij start van een sessie om te checken of RVO‑data beschikbaar en actueel is.
- Eventueel tonen in UI (laatste update, welke datasets aanwezig zijn).

### 2.2 `address_lookup`

Doel: Nederlands adres vertalen naar gebouw-/monumentinformatie via BAG, Stella Spark Nexus WFS en EP-online.

- Input (JSON): `AddressLookupInput`
  - `postal_code: str | null` (string, zonder spatie, bv. `"1012AB"`)
  - `house_number: int` (verplicht)
  - `house_number_addition: str | null` (optioneel, bv. `"A"`)
  - `street_name: str | null` – straatnaam, verplicht indien geen `postal_code`
  - `city: str | null` – plaatsnaam, verplicht indien geen `postal_code`
  
  Let op: geef óf `postal_code` óf beide `street_name` en `city`.

- Output (JSON): `AddressLookupOutput`
  - `input`: echo van de gevalideerde input
  - `timestamp`: UTC tijdstip van de response
  - `bag`: BAG lookup resultaten (address + identificaties)
    - `status`: `"ok"`, `"skipped"`, of `"error"`
    - `rd_coordinates`: RD (EPSG:28992) coördinaten `(x, y)`
    - `ids`: `nummeraanduiding_id`, `adresseerbaar_object_id`, `adresseerbaar_object_type`, `pand_ids`
    - `selected`: geselecteerde BAG record
    - `matches`: alle BAG matches
    - `error`: foutmelding indien van toepassing
  - `nexus`: Nexus lookup resultaten (gebouweenheden en monumenten)
    - `status`: `"ok"`, `"skipped"`, of `"error"`
    - `matches`: alle Nexus building_unit features
    - `selected`: geselecteerde features voor monument lookups
    - `monuments`: monument features per geselecteerde building_unit
    - `match_count`: aantal features vóór disambiguatie
    - `error`: foutmelding indien van toepassing
  - `ep_online`: EP-online energielabel lookup
    - `status`: `"ok"`, `"skipped"`, of `"error"`
    - `selected`: geselecteerde EP-online record
    - `matches`: alle EP-online matches
    - `error`: foutmelding indien van toepassing
  - `warnings`: niet-fatale waarschuwingen

### 2.3 `search_rvo_subsidies`

Doel: semantische zoektool in de RVO‑subsidiecatalogus.

- Input (JSON): `SubsidySearchInput`
  - `query: str` – vrije tekst, bv. "warmtepomp hoekwoning 2025" (min. 3 tekens)
  - `statuses: list[SubsidyStatus] | null` – filter op status (optioneel)
    - Mogelijke waarden: `"Open voor aanvragen"`, `"Bijna open voor aanvragen"`, `"Gesloten voor aanvragen"`, `"Tijdelijk gesloten voor aanvragen"`
  - `limit: int` – max aantal resultaten (1-50, default: 10)
  - `debug: bool` – diagnostische metadata toevoegen (default: `false`)
- Output (JSON): `SubsidySearchOutput`
  - `results`: lijst met subsidies (`id`, `title`, `url`, `score`, `status`)
  - `refreshed`: laatste update van de dataset
  - `debug`: diagnostische metadata (indien aangevraagd)

### 2.4 `get_rvo_subsidy`

Doel: volledige details ophalen voor een enkele RVO subsidie op basis van ID.

- Input (JSON): `SubsidyDetailInput`
  - `id: str` – stabiel ID uit zoekresultaten
- Output (JSON): `SubsidyDetailOutput`
  - `id`, `title`, `url`, `status`, `updated_at`, `categories`
  - `summary`, `intro`, `full_text` – opgeschoonde tekstvelden (geen HTML)

### 2.5 `search_rvo_meldcodes`

Doel: zoeken in meldcode‑/installatieregisters (isolatie, warmtepompen, zonneboilers, HR‑glas).

- Input (JSON): `MeldcodeSearchInput`
  - `installation_type: InstallationType` – één van:
    - `"insulation"` (isolatie)
    - `"zonneboilers"`
    - `"warmtepompen"`
    - `"hoogrendementsglas"`
    
    Let op: `"subsidies"` is **niet** toegestaan voor meldcode searches – gebruik hiervoor `search_rvo_subsidies`.
  - `queries: list[str]` – één of meer zoektermen (productnaam, merk, type…)
  - `limit: int` – max matches per query (1-10, default: 5)
  - `debug: bool` – diagnostische metadata toevoegen (default: `false`)
- Output (JSON): `MeldcodeSearchOutput`
  - `items`: dict van unieke meldcode items, geïndexeerd op ID
    - Per item: `merk`, `type`, `meldcode`, `categorie`, `subsidiabel_vermogen`, `subsidiebedrag`, `url`
  - `results`: per query een match object met `query` en `ids` (lijst van ID's die verwijzen naar `items`)
  - `refreshed`: laatste refresh timestamp
  - `debug`: diagnostische metadata (indien aangevraagd)

### 2.6 `get_meldcode_detail`

Doel: volledige details ophalen voor een enkele meldcode record op basis van ID.

- Input (JSON): `MeldcodeDetailInput`
  - `installation_type: InstallationType` – zoals hierboven
  - `id: str` – stabiel ID uit zoekresultaten
- Output (JSON): `MeldcodeDetailOutput`
  - `id`, `title`, `url`
  - `merk`, `type`, `meldcode`, `categorie`
  - `subsidiabel_vermogen`, `subsidiebedrag`
  - `koudemiddel`, `woningtype`, `minimale_dikte_mm`, `biobased`
  - `intro`, `full_text` – opgeschoonde tekstvelden (geen HTML)

#### Tools aanroepen in een client

- In programmeerbare clients (zoals de FastMCP Client) gebruik je `list_tools()` om tools te ontdekken en `call_tool()` om ze aan te roepen met JSON‑argumenten.
- In UI‑gebaseerde clients (zoals ChatGPT/Claude) mag het llm zelfstandig tools oproepen als dat nuttig is, op basis van de toolbeschrijvingen.

---

## 3. Resources – schema's en data voor de LLM

De server stelt diverse resources beschikbaar voor schema's en ondersteunende data. Belangrijke URIs:

### 3.1 `schema://analyse-schema` (AnalysisSchema)

Doel: JSON‑schema voor gestructureerde **analyse** van input (offertes, facturen, etc.).

Bevat o.a.:

- `client_metadata` – bedrijfsnaam, klantnaam, adres, datum, enz.
- Verschillende bullet‑lijsten: `core_information_bullets`, `work_description_bullets`, `financial_overview_bullets`, `installations_specifications_bullets`, `omgevingsdata_bullets`, `timeline_and_execution_bullets`
- `missing_information` – maximaal 5 vervolgvraag‑items met label, hint, etc.

Gebruik in een client:

- Lees via `resources/read` het schema in.
- Gebruik dit schema om:
  - Een `structured output` te definiëren in je LLM‑call (bij JSON‑capable modellen).
  - Uitgebreide instructies in je prompt te genereren ("volg exact dit schema").

### 3.2 `schema://report-schema` (ReportSchema / ReportEnvelope)

Doel: JSON‑schema voor de **eindrapportage** (subsidierapport).

Structuur:

- `ReportEnvelope` met:
  - `content` (`ReportContent`):
    - `huidige_offerte`
    - `installaties` (met installatieregelrijen en meldcodes)
    - `subsidie_inzichten` (landelijk/provinciaal/gemeentelijk)
    - `overige_inzichten`
    - `samenvatting_en_aanbevelingen` (topregelingen, hiaten/risico's, vervolgstappen)

Gebruik in een client:

- Als doel‑schema voor de "eind‑LLM‑call": na analyse en RVO‑search laat je de LLM exact dit schema vullen.
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

## 5. Typische end‑to‑end workflow met Sherlock MCP

Een gebruikelijke flow voor het verwerken van een offerte en subsidieonderzoek ziet er als volgt uit:

### Stap 1 – Offerteanalyse en adresonderzoek (analyse‑schema)

1. Haal de `sherlock-system` prompt op en gebruik deze als system‑prompt voor je LLM.
2. Lees `schema://analyse-schema` en gebruik dit schema als doelstructuur.
3. Laat de gebruiker één of meer offertes/facturen uploaden.
4. Laat je LLM, gestuurd door de system‑prompt en het analyse‑schema, de offerte:
   - samenvatten in de vereiste bullet‑structuur,
   - ontbrekende informatie identificeren (`missing_information`),
   - eventuele installaties en adressen in kaart brengen.
5. Gebruik de gevonden adressen om, waar relevant, de `address_lookup` tool aan te roepen en gebouw/monumentcontext aan de analyse toe te voegen.

Resultaat: een gestructureerde analyse‑JSON conform `schema://analyse-schema`, verrijkt met adrescontext.

### Stap 2 – Subsidieonderzoek op basis van de analyse

1. Gebruik de informatie uit de analyse (installaties, woningtype, context) om queries voor subsidieonderzoek op te bouwen.
2. Gebruik de tools:
   - `search_rvo_meldcodes` voor het vinden van meldcodes en productspecificaties;
   - `get_meldcode_detail` voor volledige details van een gevonden meldcode;
   - `search_rvo_subsidies` voor relevante RVO‑subsidies;
   - `get_rvo_subsidy` voor volledige details van een gevonden subsidie.
3. Lees `data://web-search/allowed-domains` en geef deze lijst als constraint mee aan je eigen web‑search‑agent of LLM, zodat alleen betrouwbare (met name Nederlandse overheid/overheid‑gerelateerde) domeinen worden gebruikt voor aanvullend onderzoek.
4. Combineer:
   - de ISDE/RVO‑data (via de tools),
   - eventuele extra web‑search resultaten binnen de toegestane domeinen,
   - en de offerte‑analyse uit stap 1
   tot één samenhangend subsidie‑beeld.

Resultaat: een verzameling gestructureerde inzichten over mogelijke regelingen, voorwaarden en relevante bronnen.

### Stap 3 – Rapportage op basis van het rapport‑schema

1. Lees `schema://report-schema` (ReportEnvelope).
2. Gebruik de system‑prompt (`sherlock-system`) opnieuw, aangevuld met:
   - de analyse‑JSON uit stap 1,
   - de subsidie‑inzichten uit stap 2,
   - eventuele aanvullende instructies (bijvoorbeeld stijl of doelgroep).
3. Laat je LLM een volledig rapport genereren dat exact voldoet aan `schema://report-schema`, inclusief:
   - beschrijving van de huidige situatie/offerte,
   - installaties‑overzicht met meldcodes en bedragen,
   - subsidie‑inzichten (landelijk/provinciaal/gemeentelijk),
   - samenvatting en aanbevelingen.
4. Gebruik het gegenereerde rapport direct in je applicatie (bijvoorbeeld als basis voor een PDF, klantpresentatie of dashboard).

In alle stappen fungeert de `sherlock-system` prompt als onderliggende systeemprompt: hij zet de rol, toon en werkwijze van de LLM. Je kunt daarbovenop extra user‑/assistant‑messages toevoegen om deze flow in een chatbot of andere gestandaardiseerde workflow te gieten.

---

## 6. Authenticatie en veiligheid

- **API key authenticatie is vereist** – alle requests moeten een geldige API key bevatten.
- Verzend de API key als Bearer token in de `Authorization` header: `Authorization: Bearer <YOUR_API_KEY>`

---

## 7. Samenvatting – hoe gebruik je deze MCP als client?

- Configureer de MCP‑client met de URL.
- Voeg de `Authorization` header toe: `Bearer <YOUR_API_KEY>`.
- Gebruik **prompts**:
  - Haal `sherlock-system` op en gebruik het als system prompt voor je LLM.
- Gebruik **resources**:
  - Lees `schema://analyse-schema` en `schema://report-schema` om je LLM‑output strikt te structureren.
  - Optioneel: gebruik `data://web-search/allowed-domains` om websearch te filteren.
- Gebruik **tools**:
  - `health_check` voor status.
  - `address_lookup` voor adres → gebouw/monument/energielabel data.
  - `search_rvo_subsidies` en `get_rvo_subsidy` voor inhoudelijke RVO‑subsidie informatie.
  - `search_rvo_meldcodes` en `get_meldcode_detail` voor meldcode/installatie informatie.

---

## 8. FastMCP Client – Sherlock MCP gebruiken vanuit Python

Naast generieke MCP‑clients (Claude Desktop, ChatGPT, etc.) kun je ook rechtstreeks vanuit Python praten met de Sherlock MCP‑server via de **FastMCP Client** (`fastmcp.Client`). Dit geeft je een simpele, programmeerbare interface om tools, resources en prompts aan te roepen.

### 8.1 Basisvoorbeeld

Onderstaand voorbeeld laat zien hoe je verbinding maakt met de Sherlock MCP‑server, de beschikbare capabilities ophaalt en een health‑check draait:

```python
import asyncio
from fastmcp import Client
from httpx import AsyncClient

async def main() -> None:
    # Configureer HTTP client met authenticatie header
    http_client = AsyncClient(
        headers={"Authorization": f"Bearer {API_KEY}"}
    )
    client = Client(MCP_URL, http_client=http_client)

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


#### 8.1.1 Tools ontdekken

Gebruik `list_tools()` om alle tools van Sherlock MCP op te halen:

```python
from fastmcp import Client

async with Client(MCP_URL, http_client=http_client) as client:
    tools = await client.list_tools()
    for tool in tools.tools:
        print("Tool:", tool.name)
        print("Beschrijving:", tool.description)
        if tool.inputSchema:
            print("Parameterschema:", tool.inputSchema)

```

Je kunt op tags filteren, bijvoorbeeld om alleen "analyse"‑tools te tonen:

```python
async with Client(MCP_URL, http_client=http_client) as client:
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

#### 8.1.2 Tools uitvoeren

Een tool roep je aan met `call_tool(name, arguments=...)`:

```python
from fastmcp import Client

async with Client(MCP_URL, http_client=http_client) as client:
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

async with Client(MCP_URL, http_client=http_client) as client:
    result = await client.call_tool(
        "search_rvo_meldcodes",
        {"installation_type": "warmtepompen", "queries": ["Nefit EnviLine 6 kW"], "limit": 3},
        timeout=5.0,
        progress_handler=my_progress_handler,
    )
```

Daarnaast kun je `meta` meesturen voor tracing of client‑informatie:

```python
async with Client(MCP_URL, http_client=http_client) as client:
    result = await client.call_tool(
        "health_check",
        {"include_resources": True},
        meta={"trace_id": "abc-123", "client": "sherlock-dashboard"},
    )
```

#### 8.1.3 Resultaten lezen (CallToolResult)

`call_tool()` retourneert een `CallToolResult` met drie belangrijke vlakken:

- `result.data` – **FastMCP‑exclusief**: volledig gehydrateerde Python‑objecten op basis van het output‑schema (incl. datetimes, enums, eigen Pydantic‑modellen, etc.).
- `result.structured_content` – de ruwe gestructureerde JSON (standaard MCP).
- `result.content` – lijst met standaard MCP content‑blocks (tekst, etc.).

Voor de Sherlock‑tools kun je in de praktijk bijna altijd `result.data` gebruiken:

```python
async with Client(MCP_URL, http_client=http_client) as client:
    result = await client.call_tool(
        "health_check",
        {"include_resources": True},
    )

    health = result.data  # reeds omgezet naar het HealthCheckOutput-model
    print("Status:", health.status)
    print("Server:", health.server_name)
    print("Timestamp:", health.timestamp)
```

#### 8.1.4 Foutafhandeling bij tools

Standaard gooit `call_tool()` een `ToolError` als de tool faalt:

```python
from fastmcp import Client
from fastmcp.exceptions import ToolError

async with Client(MCP_URL, http_client=http_client) as client:
    try:
        result = await client.call_tool("potentially_failing_tool", {"param": "value"})
        print("OK:", result.data)
    except ToolError as exc:
        print("Tool mislukt:", exc)
```

#### 8.1.5 Resources (schema's en data) via de client

Resources zijn data‑bronnen die Sherlock MCP exposeert. Voor Sherlock zijn dit met name:

- `schema://analyse-schema` – JSON‑schema voor de analysestructuur.
- `schema://report-schema` – JSON‑schema voor de rapportstructuur.
- `data://web-search/allowed-domains` – JSON met toegestane domeinen.

Je ontdekt resources met `list_resources()`:

```python
async with Client(MCP_URL, http_client=http_client) as client:
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

async with Client(MCP_URL, http_client=http_client) as client:
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

FastMCP ondersteunt ook "resource templates" (URI‑patronen met parameters) via `list_resource_templates()` en `read_resource()` met een ingevulde template‑URI. Sherlock definieert op dit moment geen templates, maar een generieke client kan hier toch mee omgaan:

```python
async with Client(MCP_URL, http_client=http_client) as client:
    templates = await client.list_resource_templates()
    print("Templates:", [t.uriTemplate for t in templates.templates])
```

In een multi‑server‑client worden URIs doorgaans geprefixt met de servernaam (conventie per client), bijvoorbeeld `sherlock:schema://analyse-schema` – controleer de FastMCP‑clientdocumentatie voor het exacte patroon.

#### 8.1.6 Prompts via de client

Prompts zijn herbruikbare prompt‑templates die door de server worden aangeboden. Sherlock heeft in ieder geval de `sherlock-system` prompt (centrale systeem‑prompt), maar de client‑API werkt generiek voor alle prompts.

**Prompts ontdekken**

```python
async with Client(MCP_URL, http_client=http_client) as client:
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
async with Client(MCP_URL, http_client=http_client) as client:
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

async with Client(MCP_URL, http_client=http_client) as client:
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
