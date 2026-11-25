# Animo MCP

Hosted [Model Context Protocol](https://modelcontextprotocol.io) server for [Animo](https://animo.co) — the events platform for teams and AI agents.

Point Cursor, Claude, Claude Code, VS Code, or any MCP client at the URL below. Agents can list companies, draft and update events, manage pages, tickets, discounts, comms and schedules, run QA, and publish — without wrapping the REST API by hand. Payments, VAT, invoicing and form ingest stay on Animo-hosted checkout.

**Endpoint:** `https://app.animo.co/mcp`
**Transport:** Streamable HTTP (`POST`)
**Docs:** [docs.animo.co/mcp](https://docs.animo.co/mcp)
**API:** [docs.animo.co](https://docs.animo.co)
**Homepage:** [animo.co](https://animo.co)

This repository is a public descriptor for directory listings (`README`, `server.json`, license). It does **not** contain Animo application source.

> Do not confuse this with the Mintlify **docs search** MCP at `https://docs.animo.co/mcp` (`npx mint-mcp add docs.animo.co`). That server only searches documentation.

## Authentication

Two supported methods (same endpoint):

1. **OAuth (recommended for chat clients)** — Claude, ChatGPT, Cursor and similar clients can connect with no pre-issued token. An unauthenticated `POST` returns `401` with a `WWW-Authenticate` header pointing at `https://app.animo.co/.well-known/oauth-protected-resource/mcp`. The client registers at `POST https://app.animo.co/oauth/register` and the user approves the `mcp:use` grant on Animo's consent screen. The agent then acts as that user; company policies still apply.
2. **Personal access token** — for clients that already hold a key, or that do not speak OAuth. Create a token in Animo: **User settings → API tokens**, select the scopes the agent needs, and copy it immediately (shown once). Tokens expire after one year. Send it on every request:

```http
POST /mcp HTTP/1.1
Host: app.animo.co
Authorization: Bearer {personal_access_token}
Accept: application/json, text/event-stream
Content-Type: application/json
```

Do not put a company identifier in connector headers unless you want a static single-workspace shortcut. Agents should call `list_companies` and pass `company_id` as a tool argument when needed.

API + MCP access is available on all Animo plans, including Free.

**Rate limits:** 120 requests/minute per access token; 30/minute per IP without a token; 10/minute per IP for `POST /oauth/register`. Responses include `X-RateLimit-Limit` and `X-RateLimit-Remaining`. Over the limit you get HTTP `429` with `Retry-After`.

## Connect

Replace `{personal_access_token}` only when you use a PAT. For OAuth, omit the `Authorization` header and complete the client's sign-in flow.

### Cursor

Add to `~/.cursor/mcp.json` (or Cursor Settings → MCP):

```json
{
    "mcpServers": {
        "animo": {
            "url": "https://app.animo.co/mcp",
            "headers": {
                "Authorization": "Bearer {personal_access_token}"
            }
        }
    }
}
```

If the client offers an OAuth / “Sign in” path for this URL, prefer that and drop the `headers` block.

### Claude Desktop (and claude.ai)

Remote connectors are added in Claude: **Customize → Connectors → Add custom connector**. Use:

- URL: `https://app.animo.co/mcp`

Complete the Animo OAuth consent when prompted. Claude reaches the server from Anthropic's cloud over HTTPS port 443 (this endpoint is public).

A PAT can be supplied as a request header (`Authorization: Bearer {personal_access_token}`) if the connector UI offers headers and you are not using OAuth.

### Claude Code

OAuth (no token in the config):

```bash
claude mcp add --transport http animo https://app.animo.co/mcp
```

Then run `/mcp` (or `claude mcp login animo`) and complete Animo sign-in.

Personal access token instead of OAuth:

```bash
claude mcp add --transport http animo https://app.animo.co/mcp \
  --header "Authorization: Bearer {personal_access_token}"
```

Do not set an `Authorization` header if you intend to use OAuth — a rejected header is treated as a failed connection, not a fall-through to sign-in.

### VS Code

Workspace file `.vscode/mcp.json` (or user MCP config via **MCP: Open User Configuration**). The top-level key is `servers`:

```json
{
    "inputs": [
        {
            "type": "promptString",
            "id": "animo-token",
            "description": "Animo personal access token (leave unused if you sign in with OAuth)",
            "password": true
        }
    ],
    "servers": {
        "animo": {
            "type": "http",
            "url": "https://app.animo.co/mcp",
            "headers": {
                "Authorization": "Bearer ${input:animo-token}"
            }
        }
    }
}
```

If VS Code offers an OAuth path for this server, you can omit `headers` and `inputs` and sign in instead. Do not commit a raw token.

## Tools

Source: official product docs at [docs.animo.co/mcp](https://docs.animo.co/mcp) and the [API changelog](https://docs.animo.co/changelog) (including Unreleased), fetched 20 Nov 2026, Europe/Amsterdam. **37 tools** are documented. The October 2026 kit listed 19 unique names.

**Not live-verified:** an unauthenticated `tools/list` on `https://app.animo.co/mcp` returns HTTP 401 (`API user unauthenticated`). A demo token is still needed to confirm `initialize` (server name/version) and the live tool list. Until then, treat the table below as documentation, not a probe of the running server.

Compared with the October 2026 listing kit: **19 tools are new** (5 discounts, 14 schedule), **18 are still present**, and **`upload_event_asset` was removed** — use `set_event_image`. Unchanged\* means the tool existed before, but current docs document extra behaviour (co-hosts, page links, or richer image guidance). There is no generic HTTP / `call_api` tool and no `set_activity_image` tool. No MCP tool deletes a discount or a code. The four schedule delete tools are destructive.

| Tool                   | Kind    | vs Oct kit  | Purpose                                                                                                                    |
| ---------------------- | ------- | ----------- | -------------------------------------------------------------------------------------------------------------------------- |
| `list_companies`       | READ    | Unchanged   | List the companies the authenticated user may use, and get ids for `company_id`.                                           |
| `list_events`          | READ    | Unchanged   | List events of the selected company. Exposes the `published` flag.                                                         |
| `get_event`            | READ    | Unchanged   | Get one event, including whether it is `published`.                                                                        |
| `create_event`         | WRITE   | Unchanged   | Create an organizing event as a draft. Never publishes and never sends mail.                                               |
| `update_event`         | WRITE   | Unchanged\* | Update event settings. Can also replace the whole co-host list (`cohosts`; `[]` clears it).                                |
| `get_event_page`       | READ    | Unchanged\* | Get the public event page, including the organization `links` list.                                                        |
| `update_event_page`    | WRITE   | Unchanged\* | Update event page content and replace the organization `links` list (`[]` clears it).                                      |
| `set_event_image`      | WRITE   | Unchanged\* | Set the event header image (`slot=header` only) from a public URL or base64.                                               |
| `list_ticket_types`    | READ    | Unchanged   | List ticket types of an event (slugs are what `publish_event` names).                                                      |
| `create_ticket_type`   | WRITE   | Unchanged   | Create a ticket type. New tickets default to hidden from the shop.                                                         |
| `update_ticket_type`   | WRITE   | Unchanged   | Update a ticket type.                                                                                                      |
| `list_discounts`       | READ    | New         | List the company's reusable discounts (paginated). Optional `event_id` also returns the first page of codes on that event. |
| `create_discount`      | WRITE   | New         | Create a company-level discount (`fixed_amount`, `percentage`, or `bundle`). Alone, it changes nothing a buyer sees.       |
| `update_discount`      | WRITE   | New         | Update a company-level discount. Set `is_active=false` to call a promotion off (no MCP delete tool).                       |
| `list_discount_codes`  | READ    | New         | List promo codes, filtered by `event_id`, `discount_id`, or both (at least one required).                                  |
| `create_discount_code` | WRITE   | New         | Create a promo code that hands a discount out on one event's paid ticket types.                                            |
| `list_comms`           | READ    | Unchanged   | List comms templates for an event.                                                                                         |
| `get_comm`             | READ    | Unchanged   | Get one comms template.                                                                                                    |
| `update_comm`          | WRITE   | Unchanged   | Update a comms template (for example `registration_confirmation`). Never sends email.                                      |
| `get_schedule`         | READ    | New         | Read the event agenda tree (timetables, locations, activities, speakers). Creates nothing.                                 |
| `create_timetable`     | WRITE   | New         | Create one day of the agenda, and an initial location so the day can hold an activity immediately.                         |
| `update_timetable`     | WRITE   | New         | Update a day's timetable.                                                                                                  |
| `delete_timetable`     | WRITE   | New         | Delete a day and that day's locations and activities. Destructive.                                                         |
| `create_location`      | WRITE   | New         | Create a stage or room within a day.                                                                                       |
| `update_location`      | WRITE   | New         | Update a location within a day.                                                                                            |
| `delete_location`      | WRITE   | New         | Delete a location and its activities. Refuses the last location on a day (use `delete_timetable`). Destructive.            |
| `create_activity`      | WRITE   | New         | Create a session in a location on a given day; can attach speakers and an image.                                           |
| `update_activity`      | WRITE   | New         | Update a session, including the whole `speakers` list and image. Cannot move it to another day.                            |
| `delete_activity`      | WRITE   | New         | Delete a session. Destructive.                                                                                             |
| `list_speakers`        | READ    | New         | List company-level speakers (reused across events). Check this before creating, to avoid duplicates.                       |
| `create_speaker`       | WRITE   | New         | Add a speaker to the company. Does not bill them on a session — attach them via activity tools.                            |
| `update_speaker`       | WRITE   | New         | Update a company-level speaker, including photo (`avatar_url` / `avatar_base64`, or `remove_avatar`).                      |
| `delete_speaker`       | WRITE   | New         | Delete a speaker and remove them from every activity on every event of the company. Destructive.                           |
| `list_attendees`       | READ    | Unchanged   | List attendees' name, email and registration status. Off by default (flag `mcp-attendee-tools` + scope `orders:read`).     |
| `get_attendee`         | READ    | Unchanged   | Get one attendee plus order, VAT and related form/lead data. Off by default (same opt-in as `list_attendees`).             |
| `qa_event`             | READ    | Unchanged   | Run publish checks on an event.                                                                                            |
| `publish_event`        | EXECUTE | Unchanged   | Make the named ticket types shop-visible (`ticket_type_slugs` required). Only when the user explicitly asks to publish.    |

Default flow: `list_companies` → `create_event` → page/image → ticket types → optional discounts, comms, schedule → `qa_event` → `publish_event`.

## Safety

- `create_event` never publishes and never sends mail.
- `publish_event` is blocked by QA errors unless the user explicitly confirms.
- Paid tickets stay on Animo-hosted checkout. Do not rebuild payments, VAT, or form ingest.
- Attendee tools are off until Animo enables them for that company.

## Official registry metadata

See [`server.json`](./server.json) (schema `2025-12-11`). Proposed name: `co.animo/animo`, remote type `streamable-http`.

## Support

- Product MCP docs: <https://docs.animo.co/mcp>
- Authentication and scopes: <https://docs.animo.co/authentication>
- API index: <https://docs.animo.co>
- TODO: public support / contact email for directory forms (confirm with Animo)

## License

The files in this descriptor repository are offered under the [MIT License](./LICENSE). Animo the product remains proprietary.
