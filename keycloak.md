# Fixing a broken login after an auth plugin (e.g. `sonar-auth-oidc`) breaks

> Researched from SonarSource docs and community threads. **Not tested.** Try it on a staging copy if you can.

## What happens

The community OIDC plugin [`vaulttec/sonar-auth-oidc`](https://github.com/vaulttec/sonar-auth-oidc) was only ever tested up to SonarQube 9.9.
SonarSource staff state that it hasn't been updated for current versions. Running it on 10.x / 2025.x / 2026.x typically leads to one of these:

| Symptom                                                                   | Typical cause                                                                                                                      |
|---------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| SonarQube doesn't start, or the container restarts in a loop. The log shows a stack trace while *installing extensions* of the plugin. | Plugin API incompatibility. The jar must go, see **Fix A**.                                                   |
| Login page is blank, keeps spinning, or the "Log in with Keycloak" button throws a 500 error. | Plugin loads but its web/login extension is broken. See **Fix A** or **B**.                                      |
| Redirect loop between SonarQube and Keycloak, or `invalid redirect_uri`. | Wrong *Server base URL*, or `X-Forwarded-Proto` is missing behind the proxy. See **Fix D**.                         |
| Nobody can log in, not even `admin`.                                      | SSO is enforced, or the local admin password is unknown. See **Fix C**.                                            |

Always look at the logs first:

```bash
docker compose logs --tail=300 sonarqube | grep -iE 'oidc|plugin|exception|error'
docker compose exec sonarqube ls -l /opt/sonarqube/extensions/plugins
```

## Fix A: remove the plugin jar (works in every case)

Removing `sonar-auth-oidc-plugin-*.jar` from `extensions/plugins` is the fix confirmed in the community.

**SonarQube container still runs:**

```bash
docker compose exec sonarqube sh -c 'rm -v /opt/sonarqube/extensions/plugins/sonar-auth-oidc-plugin-*.jar'
docker compose restart sonarqube
```

**Container crash-loops.** Edit the volume from a throwaway container:

```bash
docker compose stop sonarqube
docker run --rm -v sonarr_sonarqube_extensions:/ext alpine \
  sh -c 'ls /ext/plugins; mkdir -p /ext/disabled; mv /ext/plugins/sonar-auth-oidc-plugin-*.jar /ext/disabled/'
docker compose up -d sonarqube
```

The volume name is `<compose project>_sonarqube_extensions`. Check it with `docker volume ls`.
If you moved the jar instead of deleting it, `extensions/disabled/` gives you an easy rollback.

**Login works for an admin token but the UI doesn't.** Uninstall through the web API:

```bash
TOKEN=<admin user token>
curl -su "$TOKEN:" https://sonarqube.org/api/plugins/installed | jq -r '.plugins[] | "\(.key) \(.name) \(.version)"'
curl -su "$TOKEN:" -X POST "https://sonarqube.org/api/plugins/uninstall?key=<key from above, e.g. authoidc>"
curl -su "$TOKEN:" -X POST https://sonarqube.org/api/system/restart   # or: docker compose restart sonarqube
```

Also check that the plugin isn't re-added by your image build, an init container or provisioning. Otherwise it comes back on the next deploy.

## Fix B: disable the plugin without removing it

Use this when the plugin starts but the login screen breaks. The plugin's settings are stored in the `properties` table.

**Option 1, environment variable.** The official Docker image turns `SONAR_*` env vars into `sonar.*` properties. Properties set this way override values from the UI/DB.
Add this to the `sonarqube` service and run `docker compose up -d sonarqube`:

```yaml
    environment:
      SONAR_AUTH_OIDC_ENABLED: "false"
```

**Option 2, directly in PostgreSQL** (SonarQube stopped):

```bash
docker compose stop sonarqube
docker compose exec -T postgres psql -U sonarqube -d sonarqube <<'SQL'
SELECT prop_key, text_value FROM properties WHERE prop_key LIKE 'sonar.auth.oidc%';
UPDATE properties SET text_value = 'false', is_empty = false
 WHERE prop_key = 'sonar.auth.oidc.enabled';
SQL
docker compose start sonarqube
```

If the login page loads but only shows the SSO button, look for the "More options" / "Log in with credentials" link. You can then log in locally as `admin` and turn the provider off under *Administration → Configuration → General Settings → Authentication*.

## Fix C: regain local admin access

If nobody knows the local admin password (common once SSO "just worked"), reset `admin` to `admin` in the database.
This is the procedure SonarSource documents. It was checked against the 2026.5 `users` table, which has `crypted_password`, `salt`, `hash_method` and `reset_password`.

```bash
docker compose exec -T postgres psql -U sonarqube -d sonarqube <<'SQL'
UPDATE users
   SET crypted_password = '$2a$12$uCkkXmhW5ThVK8mpBvnXOOJRLd64LJeHTeCkSuB3lfaR2N0AYBaSi',
       salt = NULL, hash_method = 'BCRYPT', reset_password = true, active = true
 WHERE login = 'admin';
SQL
```

Log in as `admin` / `admin` right away and set a strong password. SonarQube forces the change because `reset_password = true`.
If the admin account was deleted or lost its admin rights, use the official "Reset administrator access" page for your version. It also covers re-adding `admin` to `sonar-administrators`.

Also check `sonar.forceAuthentication`. It doesn't block local login, but a misconfigured proxy or SSO header auth (`sonar.web.sso.enable`) can. Unset these temporarily if they are set.

## Fix D: proxy / base URL problems (redirect loops)

* Set *Administration → General → Server base URL* (`sonar.core.serverBaseURL`) to exactly `https://sonarqube.org`, with no trailing slash.
  Without a UI: `curl -su "$TOKEN:" -X POST 'https://sonarqube.org/api/settings/set?key=sonar.core.serverBaseURL&value=https://sonarqube.org'`
* Traefik sends `X-Forwarded-Proto: https` by default. If another proxy sits in front of it, make sure that header gets through. Otherwise SonarQube builds `http://` callback URLs.
* In Keycloak, the client's *Valid redirect URIs* must match the callback exactly. For built-in SAML that's `https://sonarqube.org/oauth2/callback/saml`.

## Long-term fix: drop the plugin and use built-in SAML with Keycloak

SonarQube Server supports SAML natively, and SonarSource documents a Keycloak setup. Generic OIDC isn't a built-in method: the 2026.5 docs list HTTP header, LDAP, SAML, GitHub, Bitbucket Cloud and GitLab.
Using built-in SAML removes the third-party plugin, and with it this whole class of upgrade breakage.

**Keycloak** (realm → *Clients → Create client*):

1. Client type **SAML**. Client ID `sonarqube`. This becomes SonarQube's *Application ID*.
2. Valid redirect URI: `https://sonarqube.org/oauth2/callback/saml`.
3. *Client scopes → dedicated scope → Mappers*:
   * `login`: User Property `username`, SAML attribute `login`
   * `name`: User Property `firstName` (or a JS/aggregate mapper for the full name), SAML attribute `name`
   * `email`: User Property `email`, SAML attribute `email`
   * optional `groups`: Group list, SAML attribute `groups`, *Full group path* off
4. Under *Keys* turn off "Client signature required", or provide an SP certificate in SonarQube and enable "Sign requests".
5. Copy the realm's IdP entity ID (`https://keycloak.company.org/realms/<realm>`), SSO URL (`.../realms/<realm>/protocol/saml`) and signing certificate. You find them in *Realm settings → SAML 2.0 Identity Provider Metadata*.

**SonarQube** (*Administration → Configuration → General Settings → Authentication → SAML → Create configuration*):

| Field                         | Value                                                   |
|-------------------------------|---------------------------------------------------------|
| Application ID                | `sonarqube`                                             |
| Provider name                 | `Keycloak` (text on the login button)                   |
| Provider ID                   | `https://keycloak.company.org/realms/<realm>`           |
| SAML login URL                | `https://keycloak.company.org/realms/<realm>/protocol/saml` |
| Identity provider certificate | the realm's RS256 signing certificate                   |
| User login / name / email attribute | `login` / `name` / `email`                        |
| Group attribute (optional)    | `groups`                                                |

Click **Test configuration**, then enable it. Keep a working local admin login (Fix C) until SAML has been confirmed by several users.

**Migrating existing OIDC users.** SonarQube links external accounts by login plus identity provider.
Users created by the OIDC plugin may show up as new accounts after switching. Check them under *Administration → Security → Users* and merge or deactivate duplicates.
Make sure the SAML `login` attribute delivers the same value the OIDC plugin used, normally the Keycloak `username`. Permissions are then easy to carry over.

## Prevention

* Before every upgrade, list third-party plugins and check their compatibility. See [upgrade.md](upgrade.md).
* Prefer built-in authentication methods over community plugins.
* Keep one break-glass local admin account with a password stored in your vault.

## Sources

* Removing the jar fixes startup: https://community.sonarsource.com/t/sonarqube-not-running-with-oidc-connect-authentication-plugin/139178
* OIDC plugin not maintained for current versions: https://community.sonarsource.com/t/request-for-reintroduction-of-oidc-protocol-support/135469
* Plugin repository and tested versions: https://github.com/vaulttec/sonar-auth-oidc
* Login broken after 10.x → 2025.1 upgrade: https://community.sonarsource.com/t/couldnt-able-to-login-to-sonar-after-upgrading-from-10-9-to-2025-1/139073
* OIDC broke after 10.4 → 10.5: https://community.sonarsource.com/t/upgrade-from-10-4-1-to-10-5-0-broke-oidc-authentication/113848
* Redirect loop on login: https://community.sonarsource.com/t/sonarqube-keeps-redirecting-back-to-login/111849
* Authentication overview (built-in methods): https://docs.sonarsource.com/sonarqube-server/instance-administration/authentication/overview/
* SAML with Keycloak: https://docs.sonarsource.com/sonarqube-server/instance-administration/authentication/saml/how-to-set-up-keycloak
* SAML overview: https://docs.sonarsource.com/sonarqube-server/instance-administration/authentication/saml/
