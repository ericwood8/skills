# Generated API tests over a live server

Read this when: you need the generated API tests of a CodeGenNew feature (ASP.NET, Python, Rust) to run over SQL Server, PostgreSQL or MySQL instead of SQLite, and Docker is not running.

The generated test fixtures read `<PROJECT>_TEST_CONNECTION` (the project name in capitals, other characters `_`). Set it and the tests use that database, with no container. They add rows and remove them again, so point it at a test copy or the standing test database, never at real data. Passwords come from the shell environment (`PGPASSWORD`, `MYSQL_PWD`), never from a file.

| Stack | Value of the variable | Run |
|---|---|---|
| ASP.NET | ADO.NET string: `Server=<host>;Database=<db>;Trusted_Connection=True;TrustServerCertificate=True`, `Host=..;Username=..;Password=..`, `Server=..;User=..;Password=..` | `dotnet test --filter "FullyQualifiedName~<Class>"` |
| Python | SQLAlchemy URL with the driver: `postgresql+psycopg://`, `mysql+pymysql://`, `mssql+pyodbc://@<server>/<db>?driver=ODBC+Driver+17+for+SQL+Server&Trusted_Connection=yes&TrustServerCertificate=yes` (the ODBC driver is the one the machine has) | `python -m pytest -q tests` in a virtual environment that holds the generated `requirements.txt` (plus `pymysql` for MySQL) |
| Rust | `postgres://` or `mysql://` URL | `python C:/Github/tools/cargo_vs.py <crate> test --test api <name>`; wrap it in `run_logged.py` |

Things that cost time:

- **Filter by class, not by project name.** A project called `LiveInbound...` puts the word in every namespace, so `~Inbound` runs the whole suite.
- **Generate the whole database, then filter.** `--table` writes no essentials and no feature routes.
- **Read stderr of the generator.** Refusals (`Error: API_X: ... needs AccessMode=Ef`) are printed there; the registration file is still written and then fails to compile, which looks like a template bug.
- **Rust runs tests in parallel threads.** Over one shared database, tests that count or delete the same rows fail at random. Prove it with `-- --test-threads=1`; a generated test that counts rows needs a lock of its own.
- **Prove the run is not a false pass.** Count `Skipped`: tests that need a secret report themselves inconclusive when the variable is missing. Count the rows left behind afterwards.
- A MySQL server on Windows stores table names in lower case; the generated code handles it, the create script should still spell them as the real database does.
- **A bit column is not a boolean on SQL Server.** SQLAlchemy writes `col IS NOT 1`, which SQL Server refuses; compare with `false()`.
- **The same API over HTTP, not only its tests:** start the generated API on a port (ASP.NET `ConnectionStrings__DbConnectionString`, Python `DATABASE_URL`, Rust `PGHOST` or `MYSQL_HOST` with the password in `PGPASSWORD` or `MYSQL_PWD`, and `PORT`), play a check script against it, stop it with `taskkill /T`. One React front end reads any of them on port 5080.

The CodeGenNew harness script `Docs/Verification/Tools/live_inbound_run.py` does all of this for the inbound landing tables; copy its structure for another feature.
