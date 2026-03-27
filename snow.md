import snowflake.connector
from cryptography.hazmat.primitives.serialization import load_pem_private_key

# Load your private key file
with open("private_key.pem", "rb") as f:
    private_key = load_pem_private_key(f.read(), password=None)

conn = snowflake.connector.connect(
    account="bcbsnc-test.privatelink",
    user="s477878",
    private_key=private_key,
    database="DEV_RPLC",
    schema="RPLC_CPC",
    # warehouse="YOUR_WAREHOUSE",  # add once you have it
    # role="YOUR_ROLE",            # add once you have it
)

cursor = conn.cursor()
cursor.execute("SELECT CURRENT_VERSION()")
print(cursor.fetchone())
conn.close()

##
pip install snowflake-connector-python cryptography


## **3. Save your private key to a file** called `private_key.pem` — it should look like:

-----BEGIN PRIVATE KEY-----
(your key content)
-----END PRIVATE KEY-----


## 4. Save the test script to a file called test_snowflake.py
python test_snowflake.py

