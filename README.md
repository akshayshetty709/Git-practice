# QA Automation – Jenkins Integration (Step-by-Step)

After the RouteIQ backend is deployed successfully by **DevOps Jenkins**, the pipeline automatically triggers the **QA Jenkins** job `Sample-QA-Automation-Job` using the Jenkins Remote Build API.

```
DevOps Jenkins                                     QA Jenkins
┌──────────────────────────────┐                   ┌─────────────────────────────┐
│ Clone → Env files → Build &  │   POST /build     │ Sample-QA-Automation-Job    │
│ switch (health check OK)     │ ───────────────►  │ Maven tests (APP_URL)       │
│ → Trigger QA Automation      │  user + API token │ → Allure, ZIP, email        │
└──────────────────────────────┘                   └─────────────────────────────┘
```

---

## 1. Who does what

| Team | Responsibility |
|------|----------------|
| **IT / Network** | Give the reachable QA Jenkins URL; open firewall/port; confirm DNS, TLS and IPv4/IPv6 |
| **QA Team** | Service account, API token, job permissions, fixed `APP_URL`, reports |
| **DevOps Team** | Store credential, update Jenkinsfile, run the trigger |
| **QA + DevOps** | Run the end-to-end test; agree how QA failures are reported |

---

## 2. Information needed before starting

Collect these first:

- [ ] QA Jenkins URL reachable from the DevOps Jenkins agent (**not** `localhost` / `127.0.0.1`), e.g. `https://qa-jenkins.company.com`
- [ ] QA Jenkins port or HTTPS endpoint
- [ ] IPv4 / IPv6 connectivity confirmation
- [ ] Source IP / hostname of the DevOps Jenkins agent
- [ ] QA service account username
- [ ] QA API token (shared via the approved secure method only)
- [ ] Exact QA job name (`Sample-QA-Automation-Job`) and folder path, if any
- [ ] Confirmation that the fixed application URL is reachable from the QA execution agent

---

## 3. Step 1 – IT / Network team

1. Confirm the DevOps Jenkins agent can reach the QA Jenkins server over the company network (a shared IP is **not** needed; each server keeps its own IP/hostname).
2. Provide the QA Jenkins URL and the port / HTTPS endpoint.
3. Allow traffic from the DevOps agent (source) to QA Jenkins (destination, port) in the firewall / security group.
4. Confirm DNS resolves the QA Jenkins hostname from the DevOps agent.
5. Confirm the DevOps agent trusts the QA Jenkins TLS certificate.
6. Verify from the DevOps agent machine:

   ```bash
   curl -I https://qa-jenkins.company.com
   ```

   A `200`, `302` or `403` response means the network path works. A timeout or DNS error means it does not.

---

## 4. Step 2 – QA Jenkins configuration (QA team)

### 4.1 Create / choose the service account
1. Log in to **QA Jenkins** as an administrator.
2. Go to **Manage Jenkins → Users** (or your SSO / security realm) and create a user for the integration, e.g. `devops-trigger`, or pick an existing one.

### 4.2 Give permissions
1. Go to **Manage Jenkins → Security → Authorization**.
2. For the service account, grant:
   - **Overall → Read**
   - **Job → Read** and **Job → Build** on `Sample-QA-Automation-Job`
3. If you use *Matrix-based security* or *Role-Based Strategy*, add the permissions there (a role on the job / folder is enough; no admin rights are needed).
4. Click **Save**.

### 4.3 Create the API token
1. Log in to QA Jenkins **as the service account**.
2. Click the user name (top right) → **Security** (or **Configure**).
3. Under **API Token**, click **Add new Token**, give it a name such as `devops-trigger`, then click **Generate**.
4. **Copy the token immediately** – it is shown only once.
5. Share it with DevOps **only** through the company-approved secure method. Do **not** paste it in chat, email, screenshots or source code.

### 4.4 Prepare the QA job
1. Open `Sample-QA-Automation-Job` → **Configure**.
2. Make sure **Disable this project** is **not** ticked.
3. In the QA Jenkinsfile, set the fixed application URL as the parameter default:

   ```groovy
   parameters {
       string(
           name: 'APP_URL',
           defaultValue: 'https://test-application.example.com',
           description: 'Fixed QA application URL'
       )
   }
   ```
4. Make sure the Maven command uses it:

   ```bash
   mvn clean test -DbaseUrl="${params.APP_URL}" -Dheadless=true
   ```
5. Verify the Java framework reads the system property `baseUrl`.
6. Keep the existing stages: Allure result validation, report generation, HTML publishing, ZIP archive and email notification.
7. Run the job manually once and confirm it passes and that the application URL is reachable from the QA execution agent.

> Because `APP_URL` has a default, the plain `/build` endpoint is enough – no parameter needs to be sent per run.

---

## 5. Step 3 – DevOps Jenkins configuration (DevOps team)

### 5.1 Add the QA credential
1. Open **DevOps Jenkins → Manage Jenkins → Credentials**.
2. Click the **(global)** domain → **Add Credentials**.
3. Fill in:

   | Field | Value |
   |-------|-------|
   | Kind | **Username with password** |
   | Scope | Global |
   | Username | QA service account (e.g. `devops-trigger`) |
   | Password | The QA **API token** |
   | ID | `qa-jenkins-api` ← must match the Jenkinsfile exactly |
   | Description | QA Jenkins remote trigger |

4. Click **Create**.

### 5.2 Update the Jenkinsfile
The updated `Jenkinsfile` already contains the changes below. Commit it to the repository branch used by the pipeline job.

**New parameters**

```groovy
string(name: 'QA_JENKINS_URL', defaultValue: 'https://qa-jenkins.example.com',
       description: 'Base URL of the QA Jenkins reachable from this agent. Do NOT use localhost.')
string(name: 'QA_JOB_NAME', defaultValue: 'Sample-QA-Automation-Job',
       description: 'Exact QA Jenkins job name. For a folder job use folder/job')
booleanParam(name: 'TRIGGER_QA', defaultValue: true,
       description: 'Trigger the QA automation job after a successful deploy')
```

**New stage (after `Build and switch`)**

```groovy
stage('Trigger QA Automation') {
    when { expression { params.TRIGGER_QA } }
    steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
            withCredentials([
                usernamePassword(
                    credentialsId: 'qa-jenkins-api',
                    usernameVariable: 'QA_USER',
                    passwordVariable: 'QA_TOKEN'
                )
            ]) {
                sh '''
                    # builds  <QA_JENKINS_URL>/job/<QA_JOB_NAME>/build  and POSTs to it
                    # credentials are passed to curl via stdin, never on the command line
                '''
            }
        }
    }
}
```

**What the stage does**
- Runs only after the deploy and health check pass.
- Rejects `localhost`, `127.0.0.1` and the sample `example.com` host.
- Builds the URL from `QA_JENKINS_URL` and `QA_JOB_NAME` (folder jobs are written as `folder/job`).
- Sends `POST .../build` with the service account and API token.
- If the trigger fails, the build is marked **UNSTABLE** (the deployment stays live; no rollback).

### 5.3 Set the real QA URL
1. Open the DevOps pipeline job and click **Build with Parameters**.
   - The new parameters appear only **after one run** with the updated Jenkinsfile. Run once (it may use the defaults), then they show up.
2. Set `QA_JENKINS_URL` to the real address from IT (e.g. `https://qa-jenkins.company.com`).
3. To avoid typing it every time, change the `defaultValue` in the Jenkinsfile and commit.

### 5.4 Agent requirements
- The agent that runs the stage needs `curl`, `awk` and `sed` (standard on Linux).
- If the agent is **Windows**, replace the `sh` block with the PowerShell version from the QA handover document.

---

## 6. Step 4 – Joint test

1. **IT / Network** confirms the DevOps agent can reach the QA Jenkins URL and port.
2. **DevOps** confirms credentials are stored securely (credential `qa-jenkins-api` exists, token not in source code).
3. **DevOps** runs the pipeline with `BRANCH` and `QA_JENKINS_URL` set; wait for deploy + health check to pass.
4. **DevOps** checks the `Trigger QA Automation` stage log. Expect:
   ```
   Triggering QA job: https://qa-jenkins.company.com/job/Sample-QA-Automation-Job/build
   QA Jenkins responded with HTTP 201
   QA trigger request submitted.
   ```
5. **QA** confirms a new `Sample-QA-Automation-Job` build appears on QA Jenkins.
6. **QA** checks the console output to confirm the fixed `APP_URL` is used.
7. **QA + DevOps** confirm the Allure report, HTML report, ZIP archive and email notification.

> Tip: tick/untick `TRIGGER_QA` to skip the QA trigger for a normal deploy.

---

## 7. Troubleshooting

| Symptom | What to check |
|---------|---------------|
| Connection timeout | Network route, firewall rule, destination port, DevOps agent source IP |
| DNS error | QA Jenkins hostname resolution from the DevOps agent |
| TLS / certificate error | The agent trusts the QA Jenkins certificate (CA in the agent's trust store) |
| `QA_JENKINS_URL is not set to the real QA Jenkins address` | Parameter still has `localhost` / `example.com` – set the real URL |
| 401 Unauthorized | Username / API token in credential `qa-jenkins-api` |
| 403 Forbidden | Service account lacks Job/Build (or Overall/Read) permission; check Jenkins security config |
| 404 Not Found | QA Jenkins URL / context path (e.g. `/jenkins`), folder path, exact job name (case-sensitive) |
| Credential error `qa-jenkins-api` not found | Credential ID typo or created in a different scope/folder |
| Build is **UNSTABLE** but deploy is fine | Only the QA trigger failed – read the `Trigger QA Automation` stage log |
| QA build starts but tests fail | Application availability, `APP_URL`, test data, browser setup, Maven console output |
| Allure report missing | Allure results directory, result JSON files, report-generation stage |

Manual test from the DevOps agent (replace the values):

```bash
curl -i -X POST -u "devops-trigger:<API_TOKEN>" \
  "https://qa-jenkins.company.com/job/Sample-QA-Automation-Job/build"
```

`HTTP/1.1 201 Created` means the trigger works.

---

## 8. Notes

- The trigger only **starts** the QA job; it does not wait for results. QA results come from QA Jenkins reports and email.
- To make QA failures fail the DevOps pipeline, the pipeline must wait for the remote job (e.g. via the Parameterized Remote Trigger plugin) – agree this between QA and DevOps first.
- Never store the API token in the repository, chat, email or screenshots.
- If the QA job is in a folder, set `QA_JOB_NAME` to `folder/job` (each segment is encoded automatically).
