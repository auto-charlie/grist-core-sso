# SSO im Fork (grist-core-sso)

Dieser Fork wird gewartet, damit **OIDC- und SAML-SSO in Self-Hosted-Grist
weiterhin funktioniert**, ohne ein Full-Grist-Activation-Key zu benötigen.

## Hintergrund

Upstream (gristlabs/grist-core) hat ab **v1.7.18** die offizielle SSO-Unterstützung
für Self-Hosted (OIDC/SAML) zurückgezogen. Aus den v1.7.18 Release Notes:

> As of this release, Grist Labs will no longer be officially supporting SSO via
> OIDC/SAML outside of the full edition of Grist. If you're already running OIDC or
> SAML on a self-hosted Grist installation configured before this change, it should
> continue to work.

Wichtig dabei: **Der Server-Code wurde nie entfernt.** Die OIDC- und SAML-Login-
Systeme werden weiterhin unverändert registriert (`app/server/lib/coreLogins.ts`,
`app/server/lib/OIDCConfig.ts`, `app/server/lib/SamlConfig.ts`) und funktionieren
vollständig, wenn sie konfiguriert sind. Was v1.7.18+ geändert hat, ist die
**Positionierung und die UI**:

| Änderung | Commit | Wirkung |
|---|---|---|
| README: OIDC/SAML aus Feature-Liste in "Features not in grist-core" | `d3b306d4` | SSO wird als Full-Edition-Feature dargestellt |
| Admin/Setup-UI: "Requires activation key"-Chip, SSO-Cards nur noch per Key-Request-Modal erreichbar | `43e1e9af` | Community-Installationen können OIDC/SAML in der UI nicht (mehr) aktivieren |
| README: SSO-Eintrag weiter zusammengeklappt | `e0444ff9` | Doku-Nachwirkung |

## Was dieser Fork ändert

Die Gating-Änderungen sind rückgängig gemacht bzw. angepasst (Basis: `v1.7.19`):

1. **`app/client/ui/AuthenticationSection.ts`** — `isMissingRequiredKey()` liefert
   jetzt immer `false`. Die einzige Stelle im gesamten Codebase, die OIDC/SAML an
   einen Activation Key koppelt (verifiziert per Codebase-Durchsuchung).
   Konsequenzen:
   - OIDC- und SAML-Cards im Setup-Wizard und Admin Panel sind wieder
     **konfigurierbar und aktivierbar** in Community-Editions.
   - Bestehende SSO-Installationen zeigen kein "Missing activation key"-Warn-Badge
     mehr.
   - Der Key-Request-Modal (`showKeyRequestModal`) existiert weiterhin, wird aber
     nicht mehr für OIDC/SAML ausgelöst (bleibt für getgrist.com-Editions-Logik intakt).
2. **`README.md` / `publiccode.yml`** — die SSO-bezogene Positionierung von v1.7.18
   wiederhergestellt (OIDC/SAML wieder in der Feature-Liste, `GRIST_LOGIN_SYSTEM_TYPE`
   wieder dokumentiert).
3. **Tests** (`test/nbrowser/AuthProvider.ts`, `QuickSetupAuth.ts`) und
   Storybook-Stories entsprechend angepasst: OIDC/SAML öffnen jetzt wieder das
   Konfigurations-Modal statt des Key-Request-Modals; keine "Requires activation
   key"-Badges.

## Konfiguration (unverändert gegenüber Upstream)

SSO wird weiterhin rein konfiguriert, nicht per UI-Lizenz:

- **OIDC:** `GRIST_OIDC_IDP_ISSUER`, `GRIST_OIDC_IDP_CLIENT_ID`,
  `GRIST_OIDC_IDP_CLIENT_SECRET`, `GRIST_OIDC_IDP_SKIP_END_SESSION_ENDPOINT`,
  `GRIST_OIDC_SP_HOST` — Details:
  [OIDC-Doku](https://support.getgrist.com/install/oidc/)
- **SAML:** `GRIST_SAML_IDP_ENTITY_ID`, `GRIST_SAML_IDP_SSO_URL`,
  `GRIST_SAML_IDP_X509_CERT`, `GRIST_SAML_IDP_METADATA`,
  `GRIST_SAML_SP_ENTITY_ID`, `GRIST_SAML_SP_SSO_URL` — Details:
  [SAML-Doku](https://support.getgrist.com/install/saml/)
- **Auswahl:** `GRIST_LOGIN_SYSTEM_TYPE=saml|oidc|forward-auth|minimal`
  (Standard: automatische Erkennung der ersten konfigurierten Methode).

## Wartungs-Verfahren (bei jedem neuen Upstream-Release)

1. `git fetch upstream` → neuen Tag prüfen.
2. **Relevanz-Check:**
   ```
   git log --oneline vVorher..vNeu -i --grep='sso\|oidc\|saml'
   git diff vVorher..vNeu --stat -- app/server/lib/oidc/ app/server/lib/SamlConfig.ts app/server/lib/OIDCConfig.ts app/server/lib/coreLogins.ts app/client/ui/AuthenticationSection.ts
   ```
3. Auf `main`: `git merge upstream/vNeu` (der Main-Branch dieses Forks trägt
   immer Upstream-Stand + unsere Patches).
4. **Nur bei Konflikten/SSO-Änderungen:** Patch-Sets 1–3 unten neu anwenden
   (bzw. per `git rebase`/`cherry-pick` der Patches).
5. **Pflicht-Verifikation** (siehe unten) vor jedem Push.

### Fork-Patches (reproduzierbar)

| Patch | Beschreibung |
|---|---|
| `sso-ui-gate-revert.patch` | UI-Gate: `isMissingRequiredKey()` → `false` (einzige funktionale Änderung) |
| `sso-docs-revert.patch` | README/publiccode SSO-Positionierung |
| `sso-tests-revert.patch` | Tests + Storybook an das neue Verhalten angepasst |

Erzeugen (nach Änderungen):
```
git diff v1.7.19 -- app/client/ui/AuthenticationSection.ts > docs/sso-ui-gate-revert.patch
```
Patches werden im `docs/`-Verzeichnis gepflegt, damit sie bei Upstream-Rebases
schnell neu angewendet werden können.

## Test-Verfahren

### Automatisierte Tests

```bash
yarn install            # postinstall baut den Python-Sandbox (erforderlich)
yarn build              # full TypeScript-Build (Typecheck enthalten)
MOCHA_WEBDRIVER_HEADLESS=1 npm run test:nbrowser -- --grep "AuthProvider|QuickSetupAuth"
```

Die nbrowser-Suiten `AuthProvider` und `QuickSetupAuth` testen:
- OIDC-Konfiguration wird erkannt (konfiguriert/fehlkonfiguriert),
- Setup-Wizard zeigt OIDC/SAML als wählbare Methoden,
- Aktivierung per UI (`POST /api/config/auth-providers/set-active`) →
  `GRIST_LOGIN_SYSTEM_TYPE` in Prefs → Login-Redirect zum IdP nach Neustart
  (inkl. echter RestartShell-Zyklen gegen einen Mock-OIDC-Issuer).

### Manuelle End-to-End-Verifikation

```bash
# Community-Edition, OIDC gegen einen eigenen IdP (z. B. Authentik/Dex/Auth0):
GRIST_LOGIN_SYSTEM_TYPE=oidc \
GRIST_OIDC_IDP_ISSUER=https://idp.example.com \
GRIST_OIDC_IDP_CLIENT_ID=... GRIST_OIDC_IDP_CLIENT_SECRET=... \
GRIST_OIDC_IDP_SKIP_END_SESSION_ENDPOINT=true \
GRIST_OIDC_SP_HOST=https://grist.example.com \
  yarn start
```
1. Grist öffnen → `/login` sollte zum IdP-Redirect führen.
2. Nach Login: User wird in der Home-DB angelegt, `Profile Settings` zeigt E-Mail/Name.
3. SAML analog mit `GRIST_LOGIN_SYSTEM_TYPE=saml` + `GRIST_SAML_IDP_METADATA=...`.
4. UI-Check: Admin Panel → Authentication zeigt OIDC als Active (grünes Badge,
   **kein** "Missing activation key"-Warnhinweis).

## Abweichungen vom Referenz-Fork

`HeroLabsAndroid/grist-core-sso-safe` verfolgt dasselbe Ziel, aber per
**Freeze auf v1.7.17** (Branch `freeze/v1.7.17`): alle Upstream-Änderungen danach
— inklusive Security-Fixes — bleiben draußen. Dieser Fork geht den
Gegenweg: aktueller Upstream + gezielte Reverts, dadurch:
- weiterhin aktuelle Security-/Bugfixes,
- SSO-Funktionalität bleibt überprüfbar (Tests laufen auf aktuellem Stand),
- Revert-Fläche ist klein und stabil (ein UI-Gate + Doku).
