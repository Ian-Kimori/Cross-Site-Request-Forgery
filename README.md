# Cross-Site-Request-Forgery

You are absolutely right. To have a defensible audit report, you must show the manual "failure" of the server's logic before you show the "automated" attack via Burp. 

Here is the **complete, end-to-end testing lifecycle** for CSRF in your Zabbix environment, structured to prove that the security gate is failing even if the user permissions (RBAC) are blocking the final action.

---

### **Phase 1: Identify the "Sensitive Action" (The Trigger)**
Before testing, you must find a request that actually modifies data.
1.  **Open Burp Suite:** Ensure "Intercept" is off but your "HTTP History" is recording.
2.  **Perform the Action:** In Zabbix, go to **Data collection > Hosts**. Select a host and click **Disable**.
3.  **Locate the Request:** In Burp, find the `POST` request to `zabbix.php?action=host.disable`.
4.  **Send to Repeater:** Right-click the request and select **Send to Repeater**. This is your "lab" for the next steps.

---

### **Phase 2: The "Badge-Only" Test (Manual Validation)**
This step proves whether the `zbx_session` cookie (The Badge) is the only thing the server truly cares about.
1.  **Baseline Run:** In Repeater, click **Send**. You should see the "Success" or "Access Denied" message (depending on your user role).
2.  **The Attack:** Go to the JSON body at the bottom and **delete** the line containing `_csrf_token`.
    * *Before:* `{"hostids":["38575"],"_csrf_token":"487fb..."}`
    * *After:* `{"hostids":["38575"]}`
3.  **Analyze the Response Code:**
    * **PASS (Secure):** The server returns **403 Forbidden** or a specific security error like "Invalid Token."
    * **FAIL (Vulnerable):** The server returns **200 OK**. Even if the message inside says "Access Denied," a `200 OK` proves the server **bypassed the security check** and tried to execute the code using only your cookie.



---

### **Phase 3: The "Token Mismatch" Test (Integrity Check)**
This proves if the server actually checks that the token belongs to *you*.
1.  **Modify the Token:** Put the `_csrf_token` back in, but change just **one character** (e.g., change the last `a` to a `b`).
2.  **Send:** Click **Send**.
3.  **Result:** If the server still returns `200 OK`, the token is "cosmetic" and not being validated against your session.

---

### **Phase 4: Use the Burp CSRF PoC Generator (The Simulation)**
This simulates how an attacker would exploit the failure found in Phase 2.
1.  **Generate:** Right-click the original request -> **Engagement tools** -> **Generate CSRF PoC**.
2.  **Options:** Click "Options" and select **Include auto-submit script**.
3.  **Save:** Click "Copy HTML" and save it as `zabbix_poc.html` on your desktop.

---

### **Phase 5: The "SameSite" & Browser Execution (The Proof)**
This is the final step to see if a real browser allows the attack.
1.  **Session Preparation:** Ensure you are logged into Zabbix in your browser.
2.  **Run the Exploit:** Double-click `zabbix_poc.html`.
3.  **Final Verification:**
    * **Check the Zabbix UI:** Did the host status change?
    * **Check the Audit Log:** Go to **Reports > Audit log**. If you see a record of the action being attempted or completed at the exact second you opened the HTML file, the test is a **FAIL**.

---

### **Phase 6: Evaluating the Results for the Report**

| Audit Step | Finding | Verdict |
| :--- | :--- | :--- |
| **Step 2 (Manual)** | Server returned `200 OK` without a token. | **FAIL** (Insecure Logic) |
| **Step 3 (Integrity)**| Server accepted a modified token. | **FAIL** (No Validation) |
| **Step 5 (Execution)**| Action appeared in Audit Log via PoC. | **FAIL** (Vulnerable to CSRF) |

### **Professional Recommendation**
To fix this, the Zabbix server must be configured to **Hard Fail** (Return 403) the moment a security token is missing or invalid. Furthermore, the `zbx_session` cookie must be set with the `SameSite=Strict` attribute, which instructs the browser never to send the cookie when the request comes from an external file like your `zabbix_poc.html`.



**One Final Question for your report:** Did you check the `Set-Cookie` header for the `SameSite` attribute? If it is missing entirely, the browser defaults to `Lax`, which is exactly why your PoC is likely to succeed.
