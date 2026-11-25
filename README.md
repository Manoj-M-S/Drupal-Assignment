# Drupal 11 Training Assignment — Recipe Hub

Drupal 11 site with a Recipe content type, Bootstrap Barrio theme + `recipe-detail` SDC, JSON:API / Views REST export, and Simple OAuth.

**Assignment write-up (all "Document" steps):** [SUBMISSION.md](SUBMISSION.md)

## What's in this repo

| Path | Purpose |
|---|---|
| `config/sync/` | Exported site configuration: Recipe type, fields, Cuisine vocabulary, View, Pathauto pattern, role, OAuth scope, enabled modules/theme |
| `web/themes/contrib/bootstrap_barrio/components/recipe-detail/` | The `recipe-detail` Single Directory Component |
| `web/themes/contrib/bootstrap_barrio/templates/content/node--recipe.html.twig` | Recipe template override |
| `web/themes/contrib/bootstrap_barrio/bootstrap_barrio.theme` | Contains the Recipe preprocess logic in `bootstrap_barrio_preprocess_node()` |
| `web/sites/default/services.yml` | Twig debug + CORS config |
| `cors-test/index.html` | Static page to test CORS from `http://localhost:3000` |
| `composer.json` / `composer.lock` | Dependencies (core, Pathauto, Simple OAuth, Bootstrap Barrio) |

Not committed: `vendor/`, core and contrib code, uploaded files, `settings.ddev.php`, OAuth keys, the client secret, and the database.

## Local setup

```bash
git clone https://github.com/Manoj-M-S/Drupal-Assignment.git && cd Drupal-Assignment
ddev start
ddev composer install
# composer install re-downloads Bootstrap Barrio; restore the customised theme files:
git checkout -- web/themes/contrib/bootstrap_barrio
ddev drush site:install --existing-config -y
```

The installed site has all the configuration but no content. Recipes, Cuisine terms, the OAuth consumer and its keys are content or secrets, so they are not in git. To get a token you also need to generate keys (`ddev drush simple-oauth:generate-keys ../private/oauth`) and create a consumer under *Configuration → Simple OAuth → Clients*. See [SUBMISSION.md](SUBMISSION.md) for details.
