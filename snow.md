import snowflake.connector
from cryptography.hazmat.primitives.serialization import (
    load_pem_private_key, Encoding, PrivateFormat, NoEncryption
)

# ── CONFIG ──────────────────────────────────────────────────────────────────
PRIVATE_KEY_PATH = "private_key.pem"
SQL_FILE_PATH    = "create_table.sql"   # developers check this file into GitHub

SNOWFLAKE_CONFIG = {
    "account":   "bcbsnc-test.privatelink",
    "user":      "s477878",
    "database":  "DEV_RPLC",
    "schema":    "RPLC_CPC",
    "warehouse": "CTO_BASE",
}
# ────────────────────────────────────────────────────────────────────────────


def load_private_key(path):
    with open(path, "rb") as f:
        p_key = load_pem_private_key(f.read(), password=None)
    return p_key.private_bytes(Encoding.DER, PrivateFormat.PKCS8, NoEncryption())


def connect(private_key_der):
    return snowflake.connector.connect(
        **SNOWFLAKE_CONFIG,
        private_key=private_key_der,
    )


def run_sql_file(cursor, path):
    with open(path, "r") as f:
        sql = f.read()

    # Support files with multiple statements separated by semicolons
    statements = [s.strip() for s in sql.split(";") if s.strip()]
    for stmt in statements:
        print(f"\n>> Executing:\n{stmt}\n")
        cursor.execute(stmt)
        print("    Success")


def verify_session(cursor):
    checks = {
        "ACCOUNT":   "SELECT CURRENT_ACCOUNT()",
        "REGION":    "SELECT CURRENT_REGION()",
        "USER":      "SELECT CURRENT_USER()",
        "ROLE":      "SELECT CURRENT_ROLE()",
        "WAREHOUSE": "SELECT CURRENT_WAREHOUSE()",
        "DATABASE":  "SELECT CURRENT_DATABASE()",
        "SCHEMA":    "SELECT CURRENT_SCHEMA()",
        "VERSION":   "SELECT CURRENT_VERSION()",
    }
    print("=== Snowflake Session Verification ===")
    for k, q in checks.items():
        cursor.execute(q)
        print(f"  {k:<12}: {cursor.fetchone()[0]}")
    print("=======================================\n")


# ── MAIN ─────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    print("Loading private key...")
    private_key_der = load_private_key(PRIVATE_KEY_PATH)

    print("Connecting to Snowflake...")
    conn = connect(private_key_der)
    cur  = conn.cursor()

    verify_session(cur)

    print(f"Reading SQL from: {SQL_FILE_PATH}")
    run_sql_file(cur, SQL_FILE_PATH)

    print("\n All statements executed successfully.")

    cur.close()
    conn.close()

    ## Crate table.sql
    -- ============================================================
-- Developer-owned SQL file
-- Each team checks their own version of this into GitHub.
-- The pipeline picks it up and executes it against Snowflake.
-- 
-- REQUIRED: Every CREATE TABLE must include the WITH TAG line.
-- ============================================================

CREATE OR REPLACE TABLE RPLC_CPC.CPC_AWS_NO_ALERTS_SENT (
    RECORD_METADATA VARIANT,
    RECORD_CONTENT  VARIANT
) WITH TAG (SF_CATALOG.DATA_SECURITY.PHI_MBR_TYPE='A');
