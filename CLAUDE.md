# Inline CRM Sites

The ASP.NET Web Forms applications of the Inline CRM platform. **Fourth in the CRM chain.**
Depends on Inline Libraries, Inline DAL Libraries and Inline CRM Reports; its two helper libraries
are consumed by CRM Applications.

```
Inline Libraries → Inline DAL Libraries → Inline CRM Reports → Inline CRM Sites → Inline CRM Applications
```

## Layout

`Inline CRM.sln` holds 5 of the 11 tracked projects:

| Project | Role |
|---|---|
| `Inline CRM Site` | **The main application.** 238 `.aspx`, 39 `.ascx`, 5 WCF `.svc`, a Web API `ReportsController`, and SignalR. |
| `Inline CRM Admin` | Platform administration, including tenant creation (`secure/account.aspx.cs`). |
| `Inline CRM Portals` | Administrator and finance-company portals, plus one WCF REST service. |
| `IDS.Site.Helper` | Shared helpers. Packed and consumed by 40 CRM Applications projects. |
| `IDS.Shared.Helper` | Shared helpers, with a ProjectReference to `IDS.Site.Helper`. Packed. |

Outside the solution, and never built by CI:

| Project | Notes |
|---|---|
| `ICRM811.Scripting.Tests` | The only tests. See *Testing*. |
| `WebhookWebAPI` | ASP.NET Core on **net8.0**, with its own `.sln`. The only modern-.NET project in the CRM estate. |
| `IDS.MailMergeHelper` | WinForms, .NET 4.6 |
| `IDS.CRM.FtpAuthentication`, `IDS.CRM.FTPServiceMonitor`, `IDS.CRM.CreateFTPEventLog` | FTP tooling on .NET 3.5, 4.7.2 and 4.6.1, last touched in 2022, with `..\..\Inline Libraries\...\bin\Debug` HintPaths |

## Multi-tenancy

One application serves every client. The **first label of the request host** selects the tenant,
which `CRMEntities.GetActiveEntityConnectionString` in Inline DAL Libraries resolves to a database
(read that repository's `CLAUDE.md`). Consequences here:

- `new CRMEntities()` is already tenant-scoped. There are 1,313 instantiations, and only 52% are
  inside a `using`. 17 are chained (`new CRMEntities().X`) and never disposed.
- **Config is checked before the database.** A connection string named for a tenant in
  `webConfigFiles/ConnectionStrings.config` silently overrides tenant resolution.
- **`http://localhost` resolves no tenant.** Browse a `local.<domain>` host so the `local`
  connection string applies.
- `CRMSharedEntitiesSA` in Admin points at the `master` catalog. It is how
  `secure/account.aspx.cs` creates tenant databases. Treat it with care.

## Configuration and secrets

Each web app's `web.config` pulls sections from `webConfigFiles/*.config` through `configSource`
(`ConnectionStrings`, `AppSettings`, `SmtpSettings`, `Log4Net`, `ServiceModelClient`).

> **The tracked config is production config, and it holds live credentials.** That includes about
> 80 connection strings with passwords, AWS access keys, SMTP and SendGrid keys, ReCaptcha and
> webhook secrets, wallet certificates, and a `machineKey` in the `web.config` of Site and Admin.
> `IDS.Common.Secrets.SecretStore` is **not** used anywhere in this repository, and
> `AWSSDK.SecretsManager` is referenced but never called. Do not add credentials, do not copy
> them, and do not echo them. `webConfigFiles/`, `WebhookWebAPI/appsettings*.json` and
> `IDS.CRM.FTPServiceMonitor/App.config` are excluded from agent reads.

Settings that look like mistakes and are load-bearing risks. Do not copy them into new code, and
do not widen them:

- All three apps disable TLS certificate validation for the whole process
  (`ServerCertificateValidationCallback` returning `true` in `Application_Start`).
- `enableViewStateMac="false"` in Site; `enableEventValidation="false"` in all three apps; the Site
  forms cookie has `requireSSL="false"`.
- `secure/reports/reportViewer.aspx.cs` passes the **resolved connection string** as a
  client-side report parameter (`ConnectionString`). This is an open issue. Pass a tenant name,
  resolved server-side.

Session state is `StateServer` at `127.0.0.1:42424`, so the **ASP.NET State Service** must be
running locally.

## Building

```
"C:\Program Files\Microsoft Visual Studio\18\Professional\MSBuild\Current\Bin\MSBuild.exe" ^
  "Inline CRM.sln" -t:Build -p:Configuration=Debug -p:Platform="Any CPU"
```

Every sibling dependency is a **NuGet package** from the org feed
(`https://nuget.pkg.github.com/InlineDataSystems/index.json`, the only source in `nuget.config`).
That covers 19 `IDS.*` ids: `IDS.Common*` and `IDS.CRM.*` from Inline Libraries, the two
`IDS.DAL.EDM.*` models, and the four report packages from CRM Reports. `packages/` has no `IDS.*`
until restored, and the restore needs a GitHub PAT with `read:packages`.

Changing an entity is a cross-repository change: edit the EDMX in Inline DAL Libraries, publish
it, then bump the version in up to five `packages.config` files here. **There is one version per
package id across the estate** (`scripts/Sync-PrivateFeed.ps1`).

Solution quirks:

- `Release|Any CPU` builds Site, Admin and Portals in **Debug**; only the two helpers get Release.
- Site's csproj lists `IDS.Site.Helper` as a ProjectReference 8 times, and Portals lists both
  helpers 8 times each. That is harmless, but do not add a ninth.

## Deployment and CI

Site, Admin and Portals deploy through `.github/workflows/deploy.yml` (GitHub Actions, then AWS
CodeDeploy to IIS on EC2 and Rackspace): `dev` -> dev, `qa` -> qa, `staging` -> staging,
`main` -> prod. **Read `Docs/DEPLOYMENT.md` before changing anything under `deploy/`.** `*.pubxml`
is still gitignored.

- `.github/workflows/pr-build.yml` restores and builds `Inline CRM.sln` on PRs, then runs
  `deploy/tests/Invoke-DeployTests.ps1` against the committed config.
- `.github/workflows/publish-nuget.yml` publishes `IDS.Site.Helper` and `IDS.Shared.Helper` on
  push to `main`, `staging`, `qa` and `dev`. The version is `yyyy.M.d.<run>`, with suffix `-s`,
  `-q` or `-d`.
- Deployed config is the committed config plus `deploy/transforms/<app>/*.xdt`: environment values
  come from `deploy/environments/<env>.json`, secrets from Secrets Manager on each server. A setting
  that differs per environment, or is secret, needs a transform entry - the build fails otherwise.
- The web apps are published in **Release** (the solution's `Release|Any CPU` maps them to Debug;
  the pipeline publishes the projects directly). The Site's Release build is x64.

## Things that will catch you out

- **Business logic lives in code-behind.** Seven files exceed 100 KB: `secure/quote.aspx.cs`
  (279 KB), `secure/filterQuote.aspx.cs` (222 KB), `secure/genericQuote.aspx.cs` (155 KB),
  `secure/sales/policy.aspx.cs` (152 KB) and `secure/resources/dialogs/addeditpayment.aspx.cs`
  (104 KB) among them. Also large: `services/secureWebService.cs` (276 KB),
  `IDS.Site.Helper/FollowupHelper.cs` (167 KB) and `services/secureRestService.svc.cs` (130 KB).
  Extract on touch rather than rewriting.
- **Pages are forked, not parameterised.** For example:
  - `quote`, `filterQuote` and `genericQuote` (and `genericQuote.js` / `filterQuote.js`) share
    hundreds of lines, and `GetEnvironmentInfo()` is copied three times;
  - `addeditvehicle`, `addEditCommVehicle` and `addeditmoto` are variants of one dialog;
  - Admin has `_Moto` and `_NonCommVeh` rate-guide pages;
  - Site's `QueryBuilder.ascx` and Admin's `QueryBuilder2.ascx` share about 700 of 800 lines;
  - Portals `secure/administrator/` and `secure/finance/` are role forks;
  - 154 relative paths exist in both Site and Admin.

  One change is usually several. Check the siblings.
- `CommandTimeout = int.MaxValue` appears 37 times. That is about 24 days, not a timeout.
- There are 54 empty `catch` blocks. Do not add more.
- 4 sync-over-async calls (`DigitalWalletPageBase.cs`, `TickerWidget.ascx.cs`). Adding `.Result`
  under the ASP.NET synchronization context risks deadlock.
- 352 `ExecuteSqlCommand` and `SqlQuery` calls, 12 of which still concatenate. Follow the VSID-002
  / ICRM-192 parameterisation pattern (see CRM Reports' `CLAUDE.md`).
- The query builders load the embedded `IDS.DAL.EDM.CRMEntities.xml` and
  `IDS.DAL.EDM.CRMSharedEntities.xml` by manifest name. They break if the DAL models are renamed.
- Tracked PDFs sit under `*/resources/temp/` and may contain customer data. They are excluded
  from agent reads. `.gitignore` covers the Site and Admin temp folders but not Portals'.
- **Encoding: CRLF, and preserve each file's BOM state.** Nearly all `.cs` files have a BOM (871 of
  889), but `.aspx` (234 of 334) and `.js` (31 of 94) are mixed. A whole-file rewrite that flips
  either turns a small diff into hundreds of lines.

## Testing

`ICRM811.Scripting.Tests` is the only test project: 190 MSTest tests on net481, for the Scripting
feature (see `Docs/ICRM-811-Scripting-Test-Plan.md`).

```
dotnet test ICRM811.Scripting.Tests
```

It restores MSTest from the org feed like everything else, so it needs the `read:packages` PAT
too; without one, restore fails with `NU1301` / 401. It is **not in the solution, so CI never
runs it**. It references no project either: it carries its
own copies of the EF entities and the `Script*` services, and its static tests read Site markup
and JS from disk. A change to the real code can therefore pass these tests without being
exercised by them. Nothing else here is tested.

## Conventions

.NET Framework 4.8.1, C# 7.3 (LangVersion unset), `packages.config` NuGet. Telerik UI for ASP.NET
AJAX 2026.3.812, Kendo UI for jQuery 2026.3.811 from the CDN, and the Telerik Reporting HTML5
viewer. `EntityDataSource` appears in 142 markup files. Server-rendered; there is no SPA
framework. Branches are ticket-keyed `ICRM-`.

## MCP servers

`.mcp.json` (Claude Code and Visual Studio Copilot) and `.vscode/mcp.json` (VS Code Copilot)
configure seven servers, matched to actual usage in this repo:

| Server | Why |
|---|---|
| `telerik-ajax-assistant` | 661 files reference `Telerik.Web.UI`; 337 of 388 pages use `<telerik:` controls |
| `kendo-jquery-assistant` | 25 files use Kendo jQuery (follow-ups, the script designer, widgets) |
| `telerik-reporting-assistant` | All three apps host the HTML5 report viewer and a `ReportsController` |
| `mssql-schema` | `CRMShared` and the tenant databases on `ia-staging-db` |
| `aws-cloudwatch` | Application logs, which reach CloudWatch through `Serilog.Sinks.AwsCloudWatch` |
| `aws-documentation` | Read-only AWS reference: S3, CloudFront, WAFv2, Transfer Family, Cognito, ElastiCache |
| `microsoft-learn` | First-party Microsoft docs (hosted, no credentials). Web Forms, WCF, EF6 and .NET Framework 4.8.1 are where model knowledge is thinnest. |

The AWS servers use the standard AWS credential chain, so set `AWS_PROFILE` as normal; nothing is
committed. The Telerik assistants need a Telerik licence. If you have none in `%AppData%\Telerik`,
run `tools/secrets/Fetch-TelerikLicense.ps1` from CRM Reports once.

### The SQL server

`mssql-schema` runs `tools/mcp/start-mssql.ps1`, which fetches the connection string for
`ia-staging-db` from SSM Parameter Store (`/inline-crm/dev-tools/mcp-readonly-db`, us-east-2) at
launch. One server covers `CRMShared` and the tenant databases; the SQL login's permissions decide
which ones it can reach. `execute_query` is on by default because these are test databases. Set
`MSSQL_ENABLE_EXECUTE_QUERY=false` for a schema-only session. The script refuses to start against
the CRM production servers listed in `$forbiddenHosts`.

Deliberately not configured: `awslabs.aws-api-mcp-server`, which can call arbitrary AWS APIs with
your credentials. Add it if you want it, knowingly.

Worth adding once authorized (they need OAuth, which cannot be done headlessly): **Atlassian
(Jira)**, since branches are `ICRM-` keyed, and **GitHub**.

## Estate-wide documents

`docs/decisions/` in the **Inline Administration Sites** repository records agent configuration
(0003) and MCP database access (0004). Both apply here. Its secret-management decision (0001,
`SecretStore`) has **not** been applied to the CRM repositories.
