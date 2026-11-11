# SEC-335 — eLECTRA Web Application Exploitation Lab

Technical journal | Preparation, connectivity, reconnaissance, and initial testing  
Status: Work completed through initial reflected-XSS input encoding test; remaining exploitation exercises not yet completed.  
Authorized scope: Course-designated eLECTRA host `192.168.77.80` only. Localhost URLs below are SSH tunnel endpoints, not separate targets.

> Evidence hygiene: Redact session cookies, passwords, and other authentication tokens from Burp screenshots before publishing this journal. Do not include credentials in a public GitHub repository. Screenshots referenced below are stored alongside this document in the companion ZIP.

## 1. Objective and environment

The objective was to establish access to the instructor-provided eLECTRA web application, configure an intercepting proxy, map basic functionality, and document observed security weaknesses. All activity described here occurred in the assigned classroom lab.

| Component | Configuration / role |
|---|---|
| Local Kali Linux VM | Browser and Burp Suite; previously observed local IP `10.0.17.42` |
| Assigned workstation | `ws06`, accessed using the assigned `tcarpenter` account |
| Workstation SSH entry point | `192.168.3.205`, TCP port `10006` |
| Classroom lab network | EasyTier connectivity available from the assigned workstation |
| eLECTRA target | `192.168.77.80`, HTTP port 80 |
| Local SSH forward | `127.0.0.1:8081` → eLECTRA `192.168.77.80:80` via `ws06` |
| Burp Suite HTTP proxy | `127.0.0.1:8080` |
| Browser | Firefox on Kali Linux |

### 1.1 EasyTier and workstation access

The assigned workstation already had EasyTier configured. I accessed `ws06` from Kali over SSH:

```bash
ssh -p 10006 tcarpenter@192.168.3.205
```

From the assigned workstation, connectivity to the classroom lab network and the eLECTRA host was established. My local Kali VM did not itself have an EasyTier interface, so directly opening the target from Kali initially produced a connection/routing error. Installing EasyTier on Kali was not required for the working solution.

> SCREENSHOT PLACEMENT — 00_easytier_connection (existing GitHub image)
> Assigned workstation EasyTier connectivity.

![EasyTier connection](https://github.com/user-attachments/assets/33d8a4ee-1681-4d22-9f25-545ed6fe91a4)

### 1.2 SSH local port forwarding

To make the lab application accessible to Firefox and Burp running on Kali, I created a local SSH tunnel:

```bash
ssh -N \
  -p 10006 \
  -L 8081:192.168.77.80:80 \
  tcarpenter@192.168.3.205
```

The terminal running this command must remain open. The `-N` option establishes forwarding without launching a remote shell. Requests to `http://127.0.0.1:8081/` are carried through SSH to `ws06` and then to eLECTRA.

Traffic flow: Firefox → Burp Suite (`127.0.0.1:8080`) → local SSH tunnel (`127.0.0.1:8081`) → `ws06` → eLECTRA (`192.168.77.80:80`).

The ports `8080` and `8081` are intentionally different: the former is Burp's listening proxy, while the latter is the SSH forwarding endpoint.

### 1.3 Burp Suite and Firefox proxy setup

I configured Burp Suite to listen on `127.0.0.1:8080` and set Firefox's manual HTTP proxy to `127.0.0.1`, port `8080`. Initially, the eLECTRA page loaded in Firefox but requests were absent from Burp HTTP history. The issue was that Firefox was bypassing the proxy for localhost traffic.

I opened `about:config` in Firefox and changed:

```text
network.proxy.allow_hijacking_localhost = true
```

After refreshing `http://127.0.0.1:8081/`, HTTP requests appeared in Burp Suite → Proxy → HTTP history, confirming interception worked. Burp's Intercept off setting was used for normal browsing; Intercept on was enabled when deliberately inspecting a request before forwarding it.

Troubleshooting: At one point an upload failed with a browser connection error to `127.0.0.1:8081`. I checked whether the local SSH listener was present:

```bash
ss -tlnp | grep 8081
```

After reestablishing the SSH tunnel, eLECTRA loaded and the upload succeeded. This error was a tunnel/connectivity issue, not evidence of server-side file rejection.

## 2. Application access and initial reconnaissance

Using the course-provided starter customer account, I authenticated to the eLECTRA online billing portal. The landing page showed a customer dashboard with account details such as registered email, connection ID, billing address, and billing-related functionality. Visible navigation included Bill Lookup, Quick Check, My Dashboard, My Account, and Upload Documents.

> SCREENSHOT PLACEMENT — 00b_electra_landing_page (existing GitHub image)
> eLECTRA customer portal landing page.

![eLECTRA landing page](https://github.com/user-attachments/assets/67b6e115-65c1-4269-baf4-7459d5cc7616)

The dashboard was used to identify functionality worth inspecting; no unauthorized account or payment actions are documented here.

## 3. Document upload reconnaissance

### 3.1 Upload page and form structure

I opened Upload Documents, at:

```http
GET /home/upload HTTP/1.1
```

The page contained two separate forms: Upload supporting document (described as a JPG/PNG meter reading or proof of payment, maximum 5 MB) and Upload profile photo. Each had its own file chooser and submit button. The page also mentioned the `/uploads` storage location and overnight malware scanning. Those descriptions are application claims; the effectiveness of malware scanning was not tested.


> SCREENSHOT PLACEMENT — 01_upload_documents.png
> Upload Documents page. Insert the image immediately below this note.

![Upload Documents page](screenshots/01_upload_documents.png)

### 3.2 Baseline harmless file upload

I created a tiny PNG test file on Kali:

```bash
printf '%s' \
  'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mP8/x8AAusB9Y9WlS8AAAAASUVORK5CYII=' \
  | base64 -d \
  > test.png
```

I selected `test.png` in the supporting-document form and submitted it while Burp was running. Burp intercepted a request with the following properties:

```http
POST /home/upload HTTP/1.1
Host: 127.0.0.1:8081
Content-Type: multipart/form-data; boundary=...

Content-Disposition: form-data;
  name="file";
  filename="test.png"
```

The form field was named `file`; the uploaded filename was `test.png`. After forwarding the request and restoring connectivity, eLECTRA displayed “Document uploaded successfully.” This established that a normal PNG was accepted, but did not demonstrate a file validation bypass or remote code execution.


> SCREENSHOT PLACEMENT — 02_burp_upload_post.png
> Burp intercepted upload POST. Insert the image immediately below this note.

![Burp intercepted upload POST](screenshots/02_burp_upload_post.png)


> SCREENSHOT PLACEMENT — 03_multipart_parameter.png
> Multipart upload field. Insert the image immediately below this note.

![Multipart upload field](screenshots/03_multipart_parameter.png)

## 4. Confirmed finding: public directory listing and unauthenticated file access

Finding classification: Information disclosure / missing access control on uploaded files.  
Affected paths: `/uploads/` and `/uploads/test.png`.  
Status: Confirmed for the tested file and directory; broader exposure of other users' files was not tested.

### 4.1 Directory indexing

I first tried `/home/uploads`, which returned an application Page Not Found response. That path was incorrect and is retained only as troubleshooting evidence (`optional_wrong_path.png`). I then opened the correct path:

```text
http://127.0.0.1:8081/uploads/
```

Apache displayed Index of /uploads, with a link to `test.png` and its file size and modification timestamp. The server footer identified Apache/2.4.67 (Debian). This showed that directory indexing was enabled for the upload location.


> SCREENSHOT PLACEMENT — 04_uploads_directory_index.png
> Public uploads directory index. Insert the image immediately below this note.

![Public uploads directory index](screenshots/04_uploads_directory_index.png)

### 4.2 Direct access to the uploaded file

Clicking the file in the directory listing opened:

```text
http://127.0.0.1:8081/uploads/test.png
```

Firefox recognized the file as a PNG and displayed it. The test image was only 1×1 pixel, so a largely blank image view was expected.


> SCREENSHOT PLACEMENT — 05_direct_png_access.png
> Direct uploaded PNG access. Insert the image immediately below this note.

![Direct uploaded PNG access](screenshots/05_direct_png_access.png)

### 4.3 Access without authentication

To test whether an authenticated session was required, I opened a new Firefox private window without signing into the customer portal. I visited the direct PNG URL and then `/uploads/`. Both were accessible: the PNG opened and the directory index displayed `test.png` without a login prompt.


> SCREENSHOT PLACEMENT — 06_private_png_access.png
> Unauthenticated PNG access. Insert the image immediately below this note.

![Unauthenticated PNG access](screenshots/06_private_png_access.png)


> SCREENSHOT PLACEMENT — 07_private_directory_index.png
> Unauthenticated directory index. Insert the image immediately below this note.

![Unauthenticated directory index](screenshots/07_private_directory_index.png)

### 4.4 Security impact and remediation

The tested upload directory is browsable without authentication, and the uploaded test file can be retrieved without authentication. Because this feature accepts payment-related supporting documents, this configuration could disclose sensitive documents if such files are placed there. No other users' documents were accessed during this test, and no claim is made that such documents were present.

Recommended remediation: Disable directory indexing (for example, Apache `Options -Indexes`); store sensitive uploads outside the publicly served web root; require authentication and per-file authorization for downloads; generate non-guessable stored filenames; and apply appropriate file-type, content, and size validation. File execution should be disabled in upload storage locations.

## 5. Quick Check endpoint reconnaissance and initial reflected-XSS testing

### 5.1 Input mapping and baseline request

I opened Quick Check — Is my connection registered? The form requested a 10-digit connection ID. Burp showed that the form submitted the ID using a URL query parameter:

```http
GET /quick/check?connection=1234567890 HTTP/1.1
Host: 127.0.0.1:8081
```

The server returned a NOT FOUND message indicating that the test connection ID was not registered. The entered value remained in the form field after submission. This identified a reflection point for testing, but not a vulnerability by itself.


> SCREENSHOT PLACEMENT — 08_quick_check_form.png
> Quick Check form. Insert the image immediately below this note.

![Quick Check form](screenshots/08_quick_check_form.png)


> SCREENSHOT PLACEMENT — 09_quick_check_get.png
> Quick Check GET request. Insert the image immediately below this note.

![Quick Check GET request](screenshots/09_quick_check_get.png)


> SCREENSHOT PLACEMENT — 10_quick_check_response.png
> Quick Check normal response. Insert the image immediately below this note.

![Quick Check normal response](screenshots/10_quick_check_response.png)

### 5.2 Harmless HTML reflection probe

I submitted a URL-encoded HTML formatting string:

```text
http://127.0.0.1:8081/quick/check?
  connection=%3Cb%3EXSSTEST%3C%2Fb%3E

# Shown across two lines for readability;
# enter as one URL in Firefox.
```

This corresponds to `<b>XSSTEST</b>`. The input field displayed the literal characters, rather than applying bold formatting. Burp's response view showed the input value encoded in the HTML attribute:

```html
<input type="text" class="form-control" name="connection"
       value="&lt;b&gt;XSSTEST&lt;/b&gt;">
```

The application encoded the `<` and `>` characters for this test. Reflected XSS was not demonstrated. Encoding of quotation marks and other contexts was proposed as a next test but had not been performed at the point this report was written.


> SCREENSHOT PLACEMENT — 11_html_reflection.png
> HTML reflection test. Insert the image immediately below this note.

![HTML reflection test](screenshots/11_html_reflection.png)


> SCREENSHOT PLACEMENT — 12_encoded_html_response.png
> Encoded HTML response. Insert the image immediately below this note.

![Encoded HTML response](screenshots/12_encoded_html_response.png)

## 6. Findings summary

| Area | Observation | Status |
|---|---|---|
| Network preparation | Assigned workstation had EasyTier access to classroom target | Completed |
| SSH forwarding | Local Kali browser reached eLECTRA through `ws06` | Completed |
| Burp interception | Firefox localhost proxy bypass resolved; HTTP requests visible | Completed |
| Portal reconnaissance | Dashboard and navigation inspected | Completed |
| PNG upload | Harmless PNG accepted via multipart POST | Completed |
| Directory indexing | `/uploads/` lists uploaded filenames and metadata | Confirmed |
| Direct file retrieval | `/uploads/test.png` accessible | Confirmed |
| Unauthenticated retrieval | File and directory accessible in private browsing | Confirmed |
| Quick Check input | `GET /quick/check?connection=...` mapped | Completed |
| Reflected XSS | HTML tag characters encoded in tested response | Not demonstrated |
| File upload RCE | No executable payload tested | Not tested |
| Other guide exercises | SQLi, IDOR, further upload tests, advanced attack chain, etc. | Not yet documented |

## 7. Conclusion and next steps

I established a working access path to the course eLECTRA application using the assigned EasyTier-enabled workstation and an SSH tunnel, then configured Firefox and Burp Suite to inspect traffic. Reconnaissance of the upload feature identified a confirmed misconfiguration: the upload directory is indexed and both the directory and my harmless uploaded PNG are accessible without authentication. This represents an information-disclosure and access-control risk for a feature designed to accept customer documents.

I also mapped the Quick Check GET parameter and performed an initial harmless HTML reflection probe. The observed response encoded the tested angle brackets, so the work to date does not prove reflected XSS. Future journal entries should record additional instructor-authorized tests, including their exact requests, responses, screenshots, security significance, and remediation. No RCE or broader compromise is claimed in this report.

---

### Screenshot sequence (in the exact order used in this report)

The numbered PNG files are in the `screenshots/` folder. The first two images are the original GitHub-hosted screenshots you supplied; their links are embedded above.

| Order | File | Section |
|---|---|---|
| 00 | EasyTier connection (GitHub link) | 1.1 |
| 00b | eLECTRA landing page (GitHub link) | 2 |
| 01 | `01_upload_documents.png` | 3.1 — Upload Documents page |
| 02 | `02_burp_upload_post.png` | 3.2 — Burp intercepted upload POST |
| 03 | `03_multipart_parameter.png` | 3.2 — Multipart upload field |
| 04 | `04_uploads_directory_index.png` | 4.1 — Public uploads directory index |
| 05 | `05_direct_png_access.png` | 4.2 — Direct uploaded PNG access |
| 06 | `06_private_png_access.png` | 4.3 — Unauthenticated PNG access |
| 07 | `07_private_directory_index.png` | 4.3 — Unauthenticated directory index |
| 08 | `08_quick_check_form.png` | 5.1 — Quick Check form |
| 09 | `09_quick_check_get.png` | 5.1 — Quick Check GET request |
| 10 | `10_quick_check_response.png` | 5.1 — Quick Check normal response |
| 11 | `11_html_reflection.png` | 5.2 — HTML reflection test |
| 12 | `12_encoded_html_response.png` | 5.2 — Encoded HTML response |

Optional troubleshooting screenshot: `optional_wrong_path.png` shows the incorrect `/home/uploads` path. It is intentionally excluded from the main screenshot sequence.
