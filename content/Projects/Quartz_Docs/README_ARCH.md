# System Architecture & Deployment SOP

**System:** Private Knowledge Base (Quartz/Obsidian)
**Owner:** Athanasios Kompouras
**Repository:** `Arthur-K-99/knowledge-base`
**Production URL:** `https://quartz.athanasioskompouras.com`
**Last Updated:** January 2026

---

## 1. High-Level Architecture
This system uses a **Static Site Generation (SSG)** pipeline secured by an **Identity-Aware Proxy (IAP)** at the edge. The content is hosted publicly on GitHub Pages but is mathematically inaccessible to the public due to Cloudflare Zero Trust policies enforcing authentication before the request reaches the origin.

* **Local Client:** Obsidian + Quartz CLI (Content Creation)
* **Version Control:** GitHub (Branch: `v4`)
* **CI/CD:** GitHub Actions (Builds & Deploys Artifacts)
* **Hosting:** GitHub Pages (Containerised Web Server)
* **Edge Security:** Cloudflare Zero Trust (Authentication Gateway)

## 2. Data Flow & Lifecycle
1.  **Drafting:** User creates Markdown content in local Obsidian vault.
2.  **Sync:** User executes the sync command via `npm`.
    * *Action:* Pulls remote changes (rebase/merge), commits local changes, and pushes to `Arthur-K-99/knowledge-base`.
3.  **Build (CI):** GitHub Actions triggers `deploy.yml`.
    * *Step 1:* Installs Node.js v22 & Dependencies.
    * *Step 2:* Runs `npx quartz build` to transpile Markdown to HTML.
    * *Step 3:* Uploads `public/` directory as a deployment artifact.
4.  **Deploy (CD):** GitHub Actions deploys the artifact to the `github-pages` environment.
5.  **Access:**
    * **Request:** `GET https://quartz.athanasioskompouras.com`
    * **Cloudflare Edge:** Intercepts request. Checks for valid JWT.
    * **Unauthenticated:** Redirects to GitHub OAuth flow.
    * **Authenticated:** Proxies request to upstream `arthur-k-99.github.io`.

## 3. Local Configuration (Work & Home)

### A. Package Scripts (`package.json`)
The `scripts` block is configured to bypass `npx` path issues on Windows and separate content syncing from software updates.

```json
"scripts": {
  "sync": "node ./quartz/bootstrap-cli.mjs sync",
  "update": "node ./quartz/bootstrap-cli.mjs update",
  "build": "npx quartz build"
}
````

- **`npm run sync`:** Backs up notes. (Pulls first, then Pushes).
    
- **`npm run update`:** Upgrades Quartz software/plugins from upstream.
    

### B. Quartz Config (`quartz.config.ts`)

- **Base URL:** `quartz.athanasioskompouras.com`
    
    - _Critical:_ Ensures internal graph links and search indexing reference the public custom domain.
        

## 4. Multi-Device Workflow (The Golden Rule)

To prevent merge conflicts between the Work Laptop and Home Device:

1. **Start of Session (The "Commute" Pull):**
    
    - _Before writing anything_, run: `npm run sync`
        
    - This fetches changes made on the other device.
        
2. **End of Session (The "Parking" Push):**
    
    - _Before closing the laptop_, run: `npm run sync`
        
    - This saves the state for the next device to pick up.
        

## 5. Edge Security Configuration

### A. Cloudflare DNS

- **Type:** `CNAME` | **Name:** `quartz` | **Target:** `arthur-k-99.github.io`
    
- **Proxy Status:** **Proxied** (Orange Cloud) - _Required for Access._
    
- **SSL/TLS Mode:** **Full (Strict)** - _Required to prevent redirect loops._
    

### B. Cloudflare Access Policy

- **Application:** Quartz Docs
    
- **Identity Provider:** GitHub
    
- **Rule Configuration:**
    
    - **Action:** Allow
        
    - **Selector:** Email
        
    - **Value:** `[YOUR_PRIMARY_GITHUB_EMAIL]` (Must match GitHub OAuth payload exactly).
        

## 6. Troubleshooting

### Issue: "Wrong Git Author / Email" on Work Device

- **Symptom:** Commits appear as `UNIONHEALTHCENTER\User` or a generic admin account.
    
- **Cause:** Enterprise environment variables or global config overriding the user identity.
    
- **Resolution:** Force local repository configuration.
    
    PowerShell
    
    ```
    git config --local user.name "Arthur-K-99"
    git config --local user.email "YOUR_PERSONAL_EMAIL"
    ```
    

### Issue: "Quartz command not found" (Windows)

- **Symptom:** `npx quartz sync` fails with "not recognized".
    
- **Cause:** `npm ci` does not create binary shims for internal source code.
    
- **Resolution:** Use `npm run sync` instead, which invokes the `bootstrap-cli.mjs` file directly via Node.
    

### Issue: "DNS check in progress" (GitHub)

- **Symptom:** GitHub Pages warning about custom domain.
    
- **Resolution:** **Ignore.** This is expected behavior when Cloudflare proxies the connection.