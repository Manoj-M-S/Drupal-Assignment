# Drupal 11 Training Assignment — Recipe Hub

**Live site (ddev share):** `<paste ddev share URL here>`
**Local URL:** https://drupal-recipe-assignment.ddev.site
**Stack:** Drupal 11.4.5 · DDEV · Bootstrap Barrio 5.5 · Pathauto · Simple OAuth 6.1

## Quick links for reviewers

| What | Path |
|---|---|
| Recipe listing (View page) | `/recipes` |
| A single Recipe (Pathauto alias) | `/recipes/margherita-pizza` |
| JSON:API collection | `/jsonapi/node/recipe` |
| Views REST export | `/api/recipes` |
| OAuth token endpoint | `POST /oauth/token` |

**OAuth consumer:** Recipe Mobile App, Client ID `recipe_mobile_app`, scope `recipe_read`, grant type `client_credentials`. The client secret is sent by DM.

---

## Part 1 — Site Building

**Content type `Recipe`**

| Field | Machine name | Type |
|---|---|---|
| Title | `title` | Default title |
| Body | `field_body` | Text (formatted, long) |
| Prep time | `field_prep_time` | Number (integer), minutes |
| Featured image | `field_featured_image` | Image |
| Category | `field_category` | Entity reference → Taxonomy, vocabulary **Cuisine** |

**Cuisine vocabulary:** Italian, North Indian, South Indian, Indian Street Food.

**Sample content (8 published recipes, 4 cuisines):**

| Recipe | Cuisine | Prep (min) |
|---|---|---|
| Vada Pav | Indian Street Food | 20 |
| White Sauce Pasta | Italian | 25 |
| Margherita Pizza | Italian | 30 |
| Masala Dose | South Indian | 60 |
| Pani Puri | Indian Street Food | 60 |
| Chole Bhature | North Indian | 60 |
| Paneer Butter Masala | North Indian | 60 |
| Idli Vada | South Indian | 100 |

**View "Recipes by Cuisine"** (`recipes_by_cuisine`)
- Page display at `/recipes`.
- Fields: featured image, title, prep time, category (cuisine).
- Filters: Content type = Recipe, Published = Yes.
- Sort: Prep time ascending.

**Contrib module: Pathauto** (with Token). I added the pattern `recipe_url`, which applies to the Recipe bundle only: `/recipes/[node:title]`. Every recipe now gets a readable URL such as `/recipes/vada-pav` instead of `/node/6`, and it sits under the same `/recipes` path as the listing.

### Structure (config) vs Content (content)

Everything I built under **Structure** is *configuration*: the Recipe content type, its four fields and their storage, the Cuisine vocabulary itself, the "Recipes by Cuisine" View, and the Pathauto pattern. It describes the *shape* of the site. It lives in config entities, can be exported to YAML with `drush config:export`, and moves between environments (dev → stage → prod) through code.

What shows up under **Content** is *content*: the eight Recipe nodes, the four Cuisine terms, and the uploaded images. That is data that editors create on each environment. It lives in entity tables in the database and is **not** part of config export. Deploying my Recipe content type to production gives an empty Recipe type; the recipes themselves have to be created, or migrated, there.

The Cuisine example shows the split clearly. The *vocabulary* "Cuisine" is config (Structure → Taxonomy), but the *terms* Italian, North Indian, and so on are content entities.

---

## Part 2 — Theme Architecture

**Contrib theme:** Bootstrap Barrio, set as the default theme.

**Theme debugging:** `twig.config.debug: true` in `web/sites/default/services.yml`.

### Template suggestion list (Recipe node, full view mode)

```html
<!-- THEME DEBUG -->
<!-- THEME HOOK: 'node' -->
<!-- FILE NAME SUGGESTIONS:
   ▪️ node--1--full.html.twig
   ▪️ node--1.html.twig
   ▪️ node--recipe--full.html.twig
   ✅ node--recipe.html.twig
   ▪️ node--full.html.twig
   ▪️ node.html.twig
-->
<!-- 💡 BEGIN CUSTOM TEMPLATE OUTPUT from 'themes/contrib/bootstrap_barrio/templates/content/node--recipe.html.twig' -->
```

Before I added the override, Drupal fell back to Barrio's `node.html.twig`. I copied that file to `templates/content/node--recipe.html.twig`, so it now targets every Recipe node in every view mode.

**Visible change:** the default `{{ content }}` output is replaced by the `recipe-detail` component. It shows the image first, then the title, then a bordered meta box with **Prep time: X min**, **Cuisine**, and **Reading time**, then the body.

### Preprocess function

`bootstrap_barrio_preprocess_node()` in `bootstrap_barrio.theme`, scoped to `$node->bundle() === 'recipe'`, adds these variables:

| Variable | What it does |
|---|---|
| `reading_time` | Strips HTML from the body, counts words, divides by 200 wpm, rounds up (minimum 1), and returns e.g. `"2 min read"` |
| `prep_time` | Integer value of `field_prep_time` (0 if empty) |
| `cuisine` | Label of the referenced Cuisine term |
| `recipe_body` | Processed (text-format-filtered) body HTML |
| `image_url` / `image_alt` | File URL generated from the image's URI, plus its alt text (falls back to the node title) |

**Why this isn't done in Twig:** Twig is a presentation layer and deliberately sandboxed. It can only call "getter" methods on entities, it has no `str_word_count()`, and it cannot call services such as `file_url_generator`. The reading-time calculation needs the raw field value, string processing, and arithmetic, and the image URL needs a service. That is business logic, which belongs in PHP. Preprocess runs once, hands the template plain, ready-to-print values, and keeps the template (and the component) simple and reusable.

### Single Directory Component `recipe-detail`

Location: `web/themes/contrib/bootstrap_barrio/components/recipe-detail/`

```
recipe-detail/
├── recipe-detail.component.yml
├── recipe-detail.twig
└── recipe-detail.css
```

**`recipe-detail.component.yml`**

```yaml
$schema: https://git.drupalcode.org/project/drupal/-/raw/HEAD/core/assets/schemas/v1/metadata.schema.json

name: Recipe detail
status: stable

props:
  type: object

  required:
    - title
    - prep_time
    - cuisine

  properties:
    title:
      type: string
      title: Recipe title

    prep_time:
      type: integer
      title: Preparation time

    cuisine:
      type: string
      title: Cuisine

    reading_time:
      type: string
      title: Reading time

    body:
      type: string
      title: Recipe body

    image_url:
      type: string
      title: Featured image URL

    image_alt:
      type: string
      title: Featured image alt text
```

The required props are `title` (string), `prep_time` (integer), and `cuisine` (string). The image is optional and split into `image_url` + `image_alt`. The Twig wraps the image in `{% if image_url %}`, so a recipe without a picture still renders correctly.

**Usage in `node--recipe.html.twig`:**

```twig
{% include 'bootstrap_barrio:recipe-detail' with {
  title: node.label,
  prep_time: prep_time|default(0),
  cuisine: cuisine|default(''),
  reading_time: reading_time|default(''),
  body: recipe_body|default(''),
  image_url: image_url|default(''),
  image_alt: image_alt|default('')
} only %}
```

SDC attaches `recipe-detail.css` automatically when the component renders, so no library definition is needed.

---

## Part 3 — Data Exposing via API

Enabled modules: `jsonapi`, `serialization`, `rest`.

### JSON:API

| Request | Result | Payload |
|---|---|---|
| `GET /jsonapi/node/recipe` | 8 recipes, all attributes + relationships | **36,940 B** |
| `?include=field_category,field_featured_image` | Same 8 recipes + `included`: 4 `taxonomy_term--cuisine`, 8 `file--file` | 56,222 B |
| `?filter[quick][condition][path]=field_prep_time&filter[quick][condition][operator]=<&filter[quick][condition][value]=30` | 2 recipes: White Sauce Pasta (25), Vada Pav (20) | 10,009 B |
| `?fields[node--recipe]=title,field_prep_time` | 8 recipes, only `title` + `field_prep_time` | **3,873 B** |

With sparse fieldsets the payload drops from 36,940 B to 3,873 B, about **90% smaller** for the same 8 records. The full response carries 15 attributes per node (`drupal_internal__nid`, `revision_timestamp`, `promote`, `sticky`, …) plus every relationship's links, none of which a listing UI needs.

### Views REST export

A second display on **Recipes by Cuisine**: REST export at `/api/recipes`, serializer format **json**. The fields use output-friendly formatters: title as plain text, cuisine as an unlinked label, image as an *Image URL*, and prep time as raw output. The fields are aliased so the output is flat:

```json
[
  {"image":"/sites/default/files/2026-08/Vadapav.webp","title":"Vada Pav","prep_time":"20","cuisine":"Indian Street Food"},
  {"image":"/sites/default/files/2026-08/PastaWhite.jpeg","title":"White Sauce Pasta","prep_time":"25","cuisine":"Italian"}
]
```

All 8 recipes come to **1,001 B**, sorted by prep time and filtered to published, reusing the View's existing filter and sort.

### Comparison

- **JSON:API** needed zero setup and is query-driven. The *client* can include relations, filter, sort, paginate, and trim fields per request, and the spec is standard. The cost is a verbose, nested envelope (`data` / `attributes` / `relationships` / `included`) that the front end has to unpack and join.
- **Views REST export** was easier to *shape* into exactly the flat JSON a consumer wants (`title`, `cuisine` name, image URL), and it gave me more server-side control over field formatting, aliases, filters, and sorting. Each new shape needs a new display, though, and the output is not a standard spec. One gotcha: with Twig debug on, rendered fields in a REST export include the debug HTML comments until you switch the formatters to plain or raw output.
- **On a real project** I'd reach for JSON:API for a decoupled front end, because one generic, spec-compliant, cache-friendly API covers every content type and new needs don't require backend work. I'd use a Views REST export for a fixed, purpose-built feed (e.g. a partner integration or a widget) where a small flat payload matters more than flexibility.

---

## Part 4 — Authentication

### Setup

1. **Simple OAuth 6.1** installed (`composer require drupal/simple_oauth`). The RSA key pair was generated outside the web root at `/var/www/html/private/oauth/{private,public}.key`, and both paths are set in *Configuration → Simple OAuth*.
2. **Role** `Recipe API Consumer` (`recipe_api_consumer`), with one permission: *View published content* (`access content`). It has no create, edit, or delete permissions.
3. **User** `recipe_app_bot`, assigned the *Recipe API Consumer* role.
4. **Scope** `recipe_read`, granularity *Role → Recipe API Consumer*, enabled for the Client Credentials grant.
5. **Consumer** *Recipe Mobile App*: Client ID `recipe_mobile_app`, confidential, grant type *Client credentials*, scope `recipe_read`, **Default user `recipe_app_bot`**, token lifetime 3600 s.
6. JSON:API was switched from read-only to "Accept all JSON:API create, read, update, and delete operations", so write attempts are decided by **permissions** rather than blocked outright with a 405.

### Get a token (client credentials)

```bash
curl -X POST https://<site>/oauth/token \
  -d grant_type=client_credentials \
  -d client_id=recipe_mobile_app \
  -d client_secret=<secret from DM> \
  -d scope=recipe_read
```

```json
{ "token_type": "Bearer", "expires_in": 3600, "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciO…" }
```

### Results

| Request | Status | Response |
|---|---|---|
| `GET /jsonapi/node/recipe` + `Authorization: Bearer <token>` | **200** | 8 recipes |
| `GET /jsonapi/node/recipe`, no token | **200** | Same 8 recipes |
| `GET /jsonapi/node/recipe` + invalid token | **401** | Unauthorized |
| `POST /jsonapi/node/recipe` + token | **403** | `{"status":"403","title":"Forbidden"}` |
| `PATCH /jsonapi/node/recipe/{uuid}` + token | **403** | `"The 'edit any recipe content' permission is required."` |

**Why the request without a token still gets 200:** the anonymous role has *View published content*, and recipes are published, so Drupal returns them to anyone. That is correct behaviour for public content. The token decides *who* you are; permissions decide *what* you may do. A request with an invalid token is rejected with 401 rather than silently treated as anonymous. The token is what's needed for anything beyond the anonymous baseline (unpublished content, writes, user-specific data), and with the limited role, writes are still refused (403).

### CORS

In `web/sites/default/services.yml`:

```yaml
cors.config:
  enabled: true
  allowedHeaders: ['authorization', 'content-type', 'accept']
  allowedMethods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS']
  allowedOrigins: ['http://localhost:3000']
  maxAge: 600
  supportsCredentials: false
```

Verified with a preflight from `http://localhost:3000`:

```
OPTIONS /jsonapi/node/recipe   (Origin: http://localhost:3000, Access-Control-Request-Headers: authorization)
HTTP/2 204
access-control-allow-origin: http://localhost:3000
access-control-allow-methods: GET, POST, PATCH, DELETE, OPTIONS
access-control-allow-headers: authorization, content-type, accept
access-control-max-age: 600
```

You can also test it in a browser. `cors-test/index.html` is a static page that fetches a token and then the recipes. Serve it with `cd cors-test && python3 -m http.server 3000` and open http://localhost:3000.

### Why cookies don't suit a decoupled front end, and how tokens fix it

Drupal's default auth is a session cookie bound to the Drupal domain. A JS app on another origin (e.g. `localhost:3000` or a mobile app) can't log in through Drupal's form, and browsers won't send that cookie cross-site without SameSite=None, `credentials: 'include'`, and credentialed CORS. Even then, cookie-authenticated writes need a CSRF token, and native apps have no cookie jar at all. With OAuth2, the client exchanges its credentials for a short-lived Bearer token and sends it explicitly in the `Authorization` header on each request. That works from any origin or device without sessions or CSRF tokens. The token is also tied to a consumer and a scope, so its permissions can be limited (read-only here) and it can be revoked independently.

---

## How to review

1. Open `<ddev share URL>/recipes` and click through to a recipe.
2. `GET <ddev share URL>/jsonapi/node/recipe?fields[node--recipe]=title,field_prep_time`
3. `GET <ddev share URL>/api/recipes`
4. Request a token with Client ID `recipe_mobile_app` and the DM'd secret, then try GET (200) and POST/PATCH (403) on `/jsonapi/node/recipe`.
