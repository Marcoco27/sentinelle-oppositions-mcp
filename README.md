# Sentinelle oppositions: French creditor objection deadlines MCP server

**MCP server and pay-per-call API for AI agents.** For any French company (SIREN), it returns the business sales, contributions, mergers, demergers and partial asset transfers published in the BODACC, the official gazette, over the last twelve months, and for each operation that opens a window to the company's creditors the exact last day to object, the number of days left and where to serve the objection.

- **MCP endpoint (streamable HTTP):** `https://oppositions.amacunia.fr/mcp`
- **REST endpoint:** `GET https://oppositions.amacunia.fr/v1/oppositions?sirens=920902343,552100554`
- **Free sample:** https://oppositions.amacunia.fr/v1/exemple
- **Docs for agents:** https://oppositions.amacunia.fr/llms.txt
- **Price:** 0.01 USDC per call (1 to 50 companies per call), paid with the [x402](https://x402.org) protocol on Base. No account, no API key, no subscription. Free trial: 3 calls per day per caller on the REST endpoint.
- **Official MCP Registry:** `io.github.Marcoco27/sentinelle-oppositions`

## Why it exists

When a French business (*fonds de commerce*) is sold, the price is frozen for **ten days** after the last publication: any creditor of the seller may serve an objection on the price, otherwise it is paid to the seller and the creditor loses that leverage (art. L141-14 Code de commerce). When the business is contributed to a company, the seller's creditors must declare their claim to the court registry within the same ten days (art. L141-22). When a company merges, demerges or transfers a branch, its creditors may object before the commercial court within **thirty days** (art. R236-11).

Nobody warns the creditors. Credit managers, debt collectors, accountants, lawyers and landlords have to watch the gazette themselves, for every customer who owes them money. This server does the watching and the date arithmetic: one call, up to 50 debtors, every operation, every open window.

## Tools

| Tool | Price | What it does |
|---|---|---|
| `oppositions_lookup` | 0.01 USDC (x402) | Business sales, mergers and open objection deadlines for 1 to 50 SIREN |
| `oppositions_example` | free | A real answer (a business sale published on 2026-11-06, computed as of 2026-11-09) to see the output shape |

**Input of `oppositions_lookup`:** `sirens`, 1 to 50 SIREN (9 digits, checksum validated) or SIRET (14 digits, reduced to the SIREN), comma-separated. Invalid input is refused and never charged.

## Example output

A shoe shop in Pau sold its business; the sale was published in the BODACC on 6 November 2026. The seller's creditors can object until 16 November 2026.

```json
{
  "date_calcul": "2026-11-09",
  "entreprises_avec_delai_ouvert": 1,
  "resultats": [{
    "siren": "920902343",
    "delais_ouverts": [{
      "nature": "vente_fonds",
      "role_de_l_entreprise": "vendeur",
      "publication_bodacc": "2026-11-06",
      "tribunal": "Greffe du Tribunal de Commerce de Pau",
      "annonce": "https://www.bodacc.fr/pages/annonces-commerciales-detail/?q.id=id:A202602141074",
      "date_limite_opposition": "2026-11-16",
      "delai_jours": 10,
      "jours_restants": 7,
      "delai_ouvert": true,
      "report_possible_jour_chome": null,
      "fondement": "art. L. 141-14 du code de commerce",
      "geste": "Former opposition au paiement du prix, par acte de commissaire de justice ou lettre recommandée…",
      "domicile_opposition": "Election de domicile au fonds vendu pour la correspondance : …",
      "journal_annonces_legales": "Mediabask"
    }],
    "nombre_avis_12_mois": 1
  }],
  "source": "BODACC, DILA (Licence Ouverte) ; délais : art. L. 141-14, L. 141-22 et R. 236-11 du code de commerce, art. 641 du code de procédure civile",
  "avertissement": "Information, pas un conseil juridique. Seule l'annonce publiée fait foi…"
}
```

Field names are in French. `delai_ouvert` is true while an objection can still be served; `geste` says what to do (in French) and `domicile_opposition` where, as published in the notice. When the company is the buyer, the operation is listed under `evenements_12_mois` without a deadline: only the seller's (or contributor's, or merging company's) creditors get a window.

## Connect

Any MCP client can connect and call `oppositions_example`. Calling `oppositions_lookup` returns an x402 *payment required* answer: the client must support x402 payments to pay it (see below).

**Claude Code**

```bash
claude mcp add --transport http sentinelle-oppositions https://oppositions.amacunia.fr/mcp
```

**Claude Desktop / claude.ai:** Settings, Connectors, *Add custom connector*, URL `https://oppositions.amacunia.fr/mcp`.

**Cursor** (`~/.cursor/mcp.json`)

```json
{ "mcpServers": { "sentinelle-oppositions": { "url": "https://oppositions.amacunia.fr/mcp" } } }
```

**VS Code** (`.vscode/mcp.json`)

```json
{ "servers": { "sentinelle-oppositions": { "type": "http", "url": "https://oppositions.amacunia.fr/mcp" } } }
```

## Paying with x402

The paid tool follows the x402 MCP convention: the first call returns the payment requirements (scheme `exact`, network Base `eip155:8453`, asset USDC, amount `10000` = 0.01 USDC); the client signs a payment, puts it in the request `_meta` under `x402/payment`, and calls again. The payment is settled only if the tool succeeds: invalid input is never charged.

A minimal Python client with the official `x402` SDK is in [`examples/pay_mcp.py`](examples/pay_mcp.py). The same service is also sold over plain HTTP (`GET /v1/oppositions`, HTTP 402 then retry), which every x402 HTTP client supports.

## Data and limits

- Source: BODACC (Bulletin officiel des annonces civiles et commerciales), published by the Direction de l'information légale et administrative (DILA), read live through its open API at each call, under the Licence Ouverte (Etalab).
- Deadline rule: the day of publication is not counted (art. 641 Code de procédure civile). Business sales: ten days after the later of the BODACC and the legal-notice newspaper dates. Mergers, demergers and partial transfers: thirty days after the BODACC notice.
- For a merger, a later insertion elsewhere may push the date back, so the date returned is the earliest possible one (`date_au_plus_tot`).
- The extension to the next working day when the deadline falls on a weekend or a public holiday (art. 642) is returned separately, in `report_possible_jour_chome`, and is not guaranteed.
- Cancelled notices open no window; correction notices never add a second, later deadline.
- Natural persons are never named. No personal data is returned or stored.
- Information only, not legal advice. The published notice prevails.

## Same author

- [Sentinelle créances](https://github.com/Marcoco27/sentinelle-creances-mcp): French insolvency notices per company and the creditor claim filing deadline.
- [Sentinelle catnat](https://github.com/Marcoco27/sentinelle-catnat-mcp): French natural-disaster decrees per commune and the open insurance-claim deadline.

## About this repository

This repository holds the public documentation, the MCP Registry manifest ([`server.json`](server.json)) and client examples. The service itself runs at `oppositions.amacunia.fr`. Issues and questions are welcome; contact: contact@amacunia.fr.

---

### En français

Serveur MCP et API payable à l'appel pour les agents IA : pour une entreprise française (SIREN), les ventes de fonds de commerce, apports, fusions et scissions publiés au BODACC, et la date limite d'opposition de ses créanciers (dix jours après une vente, trente après une fusion), avec les jours restants et l'adresse où former l'opposition. Un centime de dollar par appel (protocole x402, USDC sur Base), jusqu'à 50 entreprises. Information, pas un conseil juridique. Présentation : https://oppositions.amacunia.fr
