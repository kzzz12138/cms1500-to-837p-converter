# EMRTS Electronic Claim Entry Portal

A Django development application for capturing, validating, reviewing, and persisting CMS-1500 professional-claim information.

## CMS-1500 implementation

The application currently includes:

- a CMS-1500-oriented claim-entry form with dynamically repeatable service lines;
- normalized patient, subscriber, payer, policy, coverage, provider, diagnosis, and service-line persistence;
- NPI, ICD-10-CM, and CPT / HCPCS reference validation;
- reference source/version/update metadata visible on the dashboard;
- claim-detail reference-match indicators and audit events;
- Django Admin maintenance for operational and reference records;
- Mermaid ERD, data-model rationale, PostgreSQL-oriented schema, fictional demo data, and automated tests.

The documented reference relationships provide stable inputs for production data loaders and ANSI X12 837P integration without changing the core claim model.

## Technology

- Python 3.12+
- Django 5
- SQLite for zero-configuration development; PostgreSQL supported through environment variables
- `uv` for dependency and command execution
- Mermaid for GitHub-rendered ERD documentation

## Install and run locally

With Git and `uv` installed, run:

```bash
git clone https://github.com/ChaoLabs/emr-claim-entry-portal.git
cd emr-claim-entry-portal
uv sync
uv run python manage.py migrate
uv run python manage.py seed_sample_claims
uv run python manage.py runserver
```

Open `http://127.0.0.1:8000/`.

All seeded names, identifiers, and claims are fictional development data. Do not enter real PHI.

## Deploy to Vercel

Vercel deployment is optional and does not change the local SQLite workflow above. The deployed application uses PostgreSQL whenever `DATABASE_URL` is present.

1. Import this GitHub repository into Vercel.
2. Add a PostgreSQL integration such as Neon and connect it to the project so that Vercel provides `DATABASE_URL`.
3. Add `DJANGO_SECRET_KEY` with a newly generated secret and set `DJANGO_DEBUG=False` for Production and Preview.
4. Deploy the project. Vercel detects `config/wsgi.py`; the build script applies migrations and loads the idempotent fictional demo seed.

Generate a secret locally with:

```bash
uv run python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
```

Vercel `*.vercel.app` hosts are allowed automatically. If you add a custom domain, include it in `DJANGO_ALLOWED_HOSTS`. Use an isolated or branched PostgreSQL database for Preview deployments so their migrations and demo data do not affect Production.

The hosted instance remains a development demonstration. Do not enter real PHI or production credentials into claim fields.

## EDI 837P generation

The `edi` app turns saved claims into an ANSI X12 837P file. The file uses version `005010X222A1`. The app only writes the file to your computer. It does not send anything to a payer or a clearinghouse.

### Try it locally

Run these commands after the steps in "Install and run locally".

```bash
uv run python manage.py seed_trading_partner
uv run python tools/make_test_claims.py
uv run python manage.py generate_837p --dry-run
```

`seed_trading_partner` creates a fictional trading partner named "Demo MAC Test". Its usage indicator is `T`, so every file is a test file. You can run this command again. It updates the same record.

The demo claim `CLM-DEMO-001` from `seed_sample_claims` does not pass the checks. Its billing ZIP has 5 digits, and its tax ID has no digits. `tools/make_test_claims.py` adds two more claims. `CLM-TEST-003` passes every check. `CLM-TEST-002` fails because its payer has no payer ID.

`--dry-run` prints the file on the screen. It does not take control numbers and does not save a batch. To write a real file, run the command without `--dry-run`.

```bash
uv run python manage.py generate_837p
```

The file is saved in the current folder. The default name looks like `837p_T_000000001_20260911065214.edi`.

### Command options

| Option | What it does |
| --- | --- |
| `--partner NAME` | Uses the active trading partner with this name. You need it when more than one partner is active. |
| `--status STATUS` | Picks claims with this status. The default is `ready_for_review`. Use `--status ""` to pick claims of any status. |
| `--claim NUMBER` | Picks a claim by its claim number. You can repeat it. When you use it, `--status` is not used. |
| `--limit N` | Uses only the first N claims, ordered by creation time. |
| `--out PATH` | Writes the file to this path. |
| `--one-line` | Puts all segments on one line. By default each segment is on its own line. |
| `--ignore TAG` | Skips issues whose tag starts with TAG, for example `--ignore "2010AA N403"`. You can repeat it. |
| `--force` | Keeps claims that have issues. Claims with no coverage or no billing provider are still left out. |
| `--dry-run` | Prints the file. It does not take control numbers and does not save a batch. |

### What the command does

1. It loads the claims and collects the data for each claim.
2. It checks each claim. A claim with an issue is left out, unless you use `--ignore` or `--force`.
3. If no claim passes, it stops with an error and writes nothing.
4. It builds the segments for the claims that passed.
5. It takes the next ISA13, GS06 and ST02 numbers for the trading partner.
6. It adds the ISA, GS, ST and BHT segments, the submitter and receiver loops, and the SE, GE and IEA segments.
7. It writes the file. It also saves one `SubmissionBatch` row and one `BatchClaim` row for each claim.
8. It prints a summary. The summary shows the accepted and rejected claims, the segment count, SE01, the number of HL loops, and the issues grouped by tag.

### Checks before a claim is built

Each issue has a tag. The tag names the 837P loop and element, such as `2010AA N403`.

| Area | What is checked |
| --- | --- |
| Claim | The claim has a coverage and a billing provider. The claim total equals the sum of the service line charges. |
| Billing provider (2010AA) | The ZIP has 9 digits. The NPI has 10 digits. The tax ID has at least one digit. The street, city and state are not empty. |
| Payer (2010BB) | The payer ID, street, city and state are not empty. The ZIP has 5 or 9 digits. |
| Subscriber (2000B, 2010BA) | The relationship to the patient can be turned into a code. The member ID is not empty. The date of birth is filled in. |
| Diagnoses (2300 HI) | There is at least one diagnosis code. No code appears twice. The diagnosis order starts at 1 and has no gaps. |
| Service lines (2400) | The claim has at least one service line. Each line has a procedure code and a service date. The end date is not before the start date. Each line has at least one diagnosis pointer. Every pointer matches a diagnosis on the claim. |

### Relationship code

The app first compares the patient with the insured person. If the first name, last name, date of birth and sex are all the same, the code is `18`, which means self. If they are not the same, the app reads the "relationship to patient" text on the coverage.

| Text | Code |
| --- | --- |
| self | 18 |
| spouse | 01 |
| child, son, daughter | 19 |
| other | G8 |

Any other text cannot be turned into a code, so the claim fails the check.

### File layout

Each file has one interchange, one functional group and one transaction set.

```
ISA  GS  ST  BHT
  1000A  submitter          NM1*41, PER
  1000B  receiver           NM1*40
  2000A  billing provider   HL, PRV, NM1*85, N3, N4, REF*EI
    2000B  subscriber       HL, SBR, NM1*IL, N3, N4, DMG, NM1*PR, N3, N4
      2000C  patient        HL, PAT, NM1*QC, N3, N4, DMG
        2300  claim         CLM, HI
          2400  service line  LX, SV1, DTP*472
SE  GE  IEA
```

- Claims with the same billing provider share one 2000A loop.
- Claims with the same subscriber, payer and policy share one 2000B loop.
- The 2000C loop is added only when the patient is not the subscriber. When the patient is the subscriber, the 2300 loop sits directly under 2000B.
- Some segments, such as PRV, N3, N4, REF and NM1*82, are added only when the data is present.
- The element separator is `*`, the component separator is `:`, the segment end is `~`, and the repetition separator is `^`.
- Text is changed to upper case. Separator characters inside the data are replaced with spaces.
- Amounts drop extra zeros. For example, `125.00` becomes `125` and `88.40` becomes `88.4`.

### Database tables

| Model | What it stores |
| --- | --- |
| `TradingPartner` | The ISA and GS sender and receiver IDs, the usage indicator (`T` or `P`), and the names and IDs for the submitter and receiver loops. |
| `ControlNumber` | The last ISA13, GS06 and ST02 number used for each trading partner. Each real run adds 1 to each number. |
| `SubmissionBatch` | One row for each generated file. It keeps the file name, control numbers, claim count, segment count and total charge. A new batch has the status `generated`. |
| `BatchClaim` | One row for each claim in a batch, with its position in the file. |

These models are not in Django Admin yet. To change the demo trading partner, edit `edi/management/commands/seed_trading_partner.py` and run it again.

### Helper scripts

| Script | What it does |
| --- | --- |
| `tools/make_test_claims.py` | Adds the fictional claims `CLM-TEST-002` and `CLM-TEST-003`. Use `--remove` to delete them. Claim numbers must be unique, so run `--remove` before you add them again. |
| `tools/export_claims_csv.py` | Writes one CSV row for each service line. The column names follow the 837P elements. Each row has a ready flag and a list of issues. The default output file is `claims_837p_export.csv`. |
| `tools/build_837p_body.py` | Prints the loops from 2000A down. It does not add the envelope, BHT, or the submitter and receiver loops. It does not take control numbers. |
| `tools/test_segments.py` | Runs the tests for the `edi` package. |

Run a script with `uv run python`, for example `uv run python tools/export_claims_csv.py`.

The `sample/` folder has example 837P files for reading. This project does not create them.

## Verify

```bash
uv run python manage.py makemigrations --check
uv run python manage.py check
uv run python manage.py test claims
```

The workflow tests cover dashboard/detail rendering, dynamic capture of more than six service lines, charge aggregation, successful reference-linked claim capture, rejection of unknown reference values, and validation of CMS-1500 diagnosis pointers.

## Database documentation

- [Final CMS-1500 ERD](docs/database/cms1500-erd.md)
- [Data model and design rationale](docs/database/cms1500-data-model.md)
- [PostgreSQL-oriented schema](docs/database/schema.sql)

## Reference-data design

The demo seed loads a tiny, clearly labeled development subset. Production loaders can use official CMS source files without changing the claim schema:

- [NPPES downloadable files](https://download.cms.gov/nppes/NPI_Files.html)
- [ICD-10 files](https://www.cms.gov/medicare/coding-billing/icd-10-codes)
- [HCPCS quarterly updates](https://www.cms.gov/medicare/coding-billing/healthcare-common-procedure-system/quarterly-update)

Full CPT content requires an appropriately licensed source.
