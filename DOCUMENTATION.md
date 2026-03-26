# ID-Authentication Demo UI – Complete Technical Documentation

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture and Module Structure](#2-architecture-and-module-structure)
3. [Application Entry Point – `IdaStarter`](#3-application-entry-point--idasstarter)
4. [Main UI Controller – `IdaController`](#4-main-ui-controller--idacontroller)
   - 4.1 [Fields and Injected Dependencies](#41-fields-and-injected-dependencies)
   - 4.2 [Lifecycle Methods](#42-lifecycle-methods)
   - 4.3 [UI Initialization and State Management](#43-ui-initialization-and-state-management)
   - 4.4 [Biometric Capture Functions](#44-biometric-capture-functions)
   - 4.5 [OTP Functions](#45-otp-functions)
   - 4.6 [Authentication Request Functions](#46-authentication-request-functions)
   - 4.7 [Encryption Functions](#47-encryption-functions)
   - 4.8 [eKYC Functions](#48-ekyc-functions)
   - 4.9 [Certificate and Key Utilities](#49-certificate-and-key-utilities)
   - 4.10 [HTTP and SSL Helpers](#410-http-and-ssl-helpers)
   - 4.11 [Miscellaneous Helpers](#411-miscellaneous-helpers)
5. [Cryptography Helper – `CryptoUtility`](#5-cryptography-helper--cryptoutility)
6. [Key Management Utility – `KeyMgrUtil`](#6-key-management-utility--keymgrutil)
7. [Signature Utility – `SignatureUtil`](#7-signature-utility--signatureutil)
8. [MJPEG Stream Runner – `MjpegTestRunner`](#8-mjpeg-stream-runner--mjpegtestrunner)
9. [Data Transfer Objects (DTOs)](#9-data-transfer-objects-dtos)
   - 9.1 [BaseAuthRequestDTO](#91-baseauthrequestdto)
   - 9.2 [AuthRequestDTO](#92-authrequestdto)
   - 9.3 [AuthTypeDTO](#93-authtypedto)
   - 9.4 [RequestDTO](#94-requestdto)
   - 9.5 [OtpRequestDTO](#95-otprequestdto)
   - 9.6 [IdentityDTO](#96-identitydto)
   - 9.7 [IdentityInfoDTO](#97-identityinfodto)
   - 9.8 [BioIdentityInfoDTO](#98-bioidentityinfodto)
   - 9.9 [DataDTO](#99-datadto)
   - 9.10 [EncryptionRequestDto](#910-encryptionrequestdto)
   - 9.11 [EncryptionResponseDto](#911-encryptionresponsedto)
   - 9.12 [CryptomanagerRequestDto](#912-cryptomanagerrequestdto)
   - 9.13 [CertificateChainResponseDto](#913-certificatechainresponsedto)
   - 9.14 [JWTSignatureRequestDto](#914-jwtsignaturerequestdto)
   - 9.15 [JWTSignatureResponseDto](#915-jwtsignatureresponsedto)
10. [Configuration and Properties](#10-configuration-and-properties)
11. [End-to-End Authentication Flow](#11-end-to-end-authentication-flow)

---

## 1. Project Overview

The **ID-Authentication Demo UI** is a JavaFX desktop application that demonstrates the ID-Authentication services offered by the [MOSIP (Modular Open Source Identity Platform)](https://mosip.io) platform. It acts as a **Partner Application** that:

- Collects biometric samples (fingerprint, iris, face) from a compliant SBI (Secure Biometric Interface) device via local HTTP calls.
- Composes and encrypts authentication requests conforming to the MOSIP IDA (ID-Authentication) REST API.
- Signs every outgoing request using a partner private key (RS256 / JWT).
- Submits authentication requests to the MOSIP IDA server and displays the result.
- Supports eKYC (Know Your Customer) authentication where the encrypted KYC response is decrypted and displayed.

### Supported Authentication Scenarios

| # | Scenario |
|---|----------|
| 1 | Single Fingerprint Authentication |
| 2 | Multiple Fingerprint Authentication |
| 3 | Single Iris Authentication |
| 4 | Multiple Iris Authentication |
| 5 | Face Authentication |
| 6 | OTP Generation + Authentication |
| 7 | Demographic (JSON) Authentication |
| 8 | Multi-modal Biometrics (Finger + Iris + Face) |
| 9 | Multi-factor (OTP + Biometrics) |
| 10 | eKYC Authentication |

---

## 2. Architecture and Module Structure

```
authentication-demo-ui/
└── src/main/java/io/mosip/authentication/demo/
    ├── dto/                  ← Plain data transfer objects (Lombok @Data)
    │   ├── AuthRequestDTO
    │   ├── AuthTypeDTO
    │   ├── BaseAuthRequestDTO
    │   ├── BioIdentityInfoDTO
    │   ├── CertificateChainResponseDto
    │   ├── CryptomanagerRequestDto
    │   ├── DataDTO
    │   ├── EncryptionRequestDto
    │   ├── EncryptionResponseDto
    │   ├── IdentityDTO
    │   ├── IdentityInfoDTO
    │   ├── JWTSignatureRequestDto
    │   ├── JWTSignatureResponseDto
    │   ├── OtpRequestDTO
    │   └── RequestDTO
    ├── helper/
    │   └── CryptoUtility      ← AES/RSA encryption wrapper
    └── service/
        ├── IdaStarter         ← Application entry point (Spring Boot + JavaFX)
        ├── IdaController      ← JavaFX FXML controller + auth orchestration
        ├── MjpegTestRunner    ← Background MJPEG stream reader
        └── util/
            ├── KeyMgrUtil     ← X.509 PKI / PKCS12 key store management
            └── SignatureUtil  ← JWS (RS256) request signing
```

The application is built on **Spring Boot** (for dependency injection and configuration) with a **JavaFX** UI layer. The FXML layout file (`fxml/idaFXML.fxml`) defines the visual components that are wired to controller fields annotated with `@FXML`.

---

## 3. Application Entry Point – `IdaStarter`

**Package:** `io.mosip.authentication.demo.service`  
**Extends:** `javafx.application.Application`  
**Annotation:** `@SpringBootApplication`, `@Import({CryptoCore.class, CryptoUtility.class})`

`IdaStarter` bridges Spring Boot's application context with the JavaFX lifecycle. It is the class whose `main` method is invoked to start the entire application.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `context` | `ConfigurableApplicationContext` | The Spring application context. Holds all Spring beans. |
| `loader` | `FXMLLoader` | JavaFX FXML loader used to parse the UI definition file. |
| `root` | `GridPane` | Root layout node parsed from the FXML file. |

### Methods

#### `main(String[] args)`
**Signature:** `public static void main(String[] args)`

The standard Java entry point. Delegates immediately to `Application.launch(IdaStarter.class, args)`, which triggers the JavaFX toolkit and calls `init()` followed by `start()`.

---

#### `init()`
**Signature:** `public void init() throws Exception`  
**Override of:** `javafx.application.Application#init()`

Called by the JavaFX runtime before the window is shown. Performs the following steps:

1. Creates a `SpringApplicationBuilder` using `IdaStarter` as the configuration class and starts the Spring context, passing any command-line parameters as Spring properties.
2. Sets the `FXMLLoader`'s controller factory to `context::getBean` so that JavaFX will obtain FXML controllers directly from the Spring container (enabling `@Autowired` injection in controllers).
3. Loads the FXML layout from the classpath resource `fxml/idaFXML.fxml` into `root`.
4. Closes the Spring context after loading (beans have already been injected at this point).

---

#### `start(Stage stage)`
**Signature:** `public void start(Stage stage) throws IOException`  
**Override of:** `javafx.application.Application#start(Stage)`

Called by the JavaFX runtime with the primary stage. Wraps `root` in a `Scene`, sets the scene on the stage, sets the title to `"ID-Authentication"`, and calls `stage.show()` to display the window.

---

## 4. Main UI Controller – `IdaController`

**Package:** `io.mosip.authentication.demo.service`  
**Annotation:** `@Component`

`IdaController` is both the JavaFX FXML controller (all `@FXML`-annotated fields are bound to UI controls) and the core authentication orchestrator. It coordinates biometric capture, request construction, encryption, signing, and submission to the MOSIP IDA server.

---

### 4.1 Fields and Injected Dependencies

#### Spring-Injected Fields

| Field | Type | Description |
|-------|------|-------------|
| `env` | `Environment` | Spring `Environment` for reading application properties at runtime. |
| `signRefid` | `String` | Value of property `mosip.sign.refid` (default `"SIGN"`). Used as signing reference ID. |
| `signatureUtil` | `SignatureUtil` | Utility for creating JWS signatures on outgoing requests. |
| `keyMgrUtil` | `KeyMgrUtil` | Utility for managing PKCS12 key stores and X.509 certificates. |
| `cryptoUtil` | `CryptoUtility` | Utility for AES symmetric and RSA asymmetric encryption. |

#### FXML-Bound UI Controls

| Field | JavaFX Type | Purpose |
|-------|-------------|---------|
| `parent` | `GridPane` | Root layout pane. |
| `irisCount` | `ComboBox<String>` | Selects Left/Right/Both Iris. |
| `fingerCount` | `ComboBox<String>` | Selects number of fingers (1–10). |
| `idValue` | `TextField` | User types the individual ID (UIN/VID/USERID). |
| `fingerAuthType` | `CheckBox` | Enables fingerprint authentication. |
| `irisAuthType` | `CheckBox` | Enables iris authentication. |
| `faceAuthType` | `CheckBox` | Enables face authentication. |
| `otpAuthType` | `CheckBox` | Enables OTP authentication. |
| `demoAuthType` | `CheckBox` | Enables demographic authentication. |
| `eKyc` | `CheckBox` | Selects eKYC mode (uses KYC endpoint instead of Auth). |
| `demoInputData` | `TextArea` | JSON input for demographic data. |
| `idTypebox` | `ComboBox<String>` | Selects ID type (UIN, VID, USERID). |
| `otpValue` | `TextField` | OTP entered by user. |
| `otpAnchorPane` | `AnchorPane` | Panel containing OTP input (enabled/disabled based on `otpAuthType`). |
| `bioAnchorPane` | `AnchorPane` | Panel containing biometric capture controls. |
| `responsetextField` | `TextField` | Displays status/result messages to user. |
| `img` | `ImageView` | Reserved for displaying images (MJPEG stream). |
| `requestOtp` | `Button` | Triggers OTP generation request. |
| `sendAuthRequest` | `Button` | Triggers the authentication request. |
| `ekycResponse` | `Button` | Shown after successful eKYC; opens eKYC data dialog. |

#### Instance Variables

| Field | Type | Description |
|-------|------|-------------|
| `capture` | `String` | JSON string of the most recent biometric capture result (combined from all modalities). |
| `previousHash` | `String` | The `hash` value returned by SBI from the last biometric capture segment; used as `previousHash` in the next capture request. |
| `ekycResponseText` | `String` | Pretty-printed JSON of the decrypted eKYC response. |
| `dialog` | `Stage` | The eKYC response popup window (created lazily). |
| `ekycTextArea` | `TextArea` | The text area inside the eKYC popup. |
| `mapper` / `objectMapper` | `ObjectMapper` | Jackson JSON serializer/deserializer instances. |

---

### 4.2 Lifecycle Methods

#### `postConstruct()`
**Signature:** `public void postConstruct() throws ...`  
**Annotation:** `@PostConstruct`

Called by Spring after the bean is fully initialized. Delegates to `initializeKeysAndCerts()` to ensure that the partner's PKI key material is ready before the UI is shown.

---

#### `initializeKeysAndCerts()`
**Signature:** `private void initializeKeysAndCerts() throws ...`

Reads `partnerId` and `partnerOrg` from application properties and determines the `keys/` directory path. It calls `keyMgrUtil.getKeyEntry(dirPath, partnerId)` to check whether a partner PKCS12 key file already exists:

- **If no key file exists:** calls `keyMgrUtil.getPartnerCertificates(...)` to auto-generate a CA, Intermediate, and Partner key pair and certificate chain.
- **If key file exists:** logs that keys are already available and skips generation.

This ensures the application always has a private key and certificate for request signing before any UI interaction.

---

#### `getKeysDirPath()`
**Signature:** `private String getKeysDirPath()`

Returns the absolute path to the `keys` directory, resolved relative to the current working directory: `<CWD>/keys`.

---

### 4.3 UI Initialization and State Management

#### `initialize()`
**Signature:** `private void initialize()`  
**Annotation:** `@FXML` (called automatically by JavaFX after FXML loading)

Sets up the initial state of all UI controls:

1. Clears `responsetextField`.
2. Populates the `idTypebox` combo box with `["UIN", "VID", "USERID"]` and selects `"UIN"`.
3. Populates `fingerCount` with `["1"…"10"]` and selects `"1"`.
4. Populates `irisCount` with `["Left Iris", "Right Iris", "Both Iris"]` and selects the first item.
5. Disables `otpAnchorPane`, `bioAnchorPane`, `demoInputData`, `requestOtp`, `sendAuthRequest`, and `responsetextField` so no action can be taken before an ID is entered.
6. Hides `ekycResponse` button.
7. Attaches a listener on `idValue.textProperty()` to call `updateSendButton()` whenever the ID field changes.
8. Attaches a listener on `otpValue.textProperty()` to call `updateSendButton()` whenever the OTP field changes.
9. Wires the `ekycResponse` button action to `displayDialog(ekycResponseText)`.

---

#### `displayDialog(String data)`
**Signature:** `private void displayDialog(String data)`

Creates (lazily, on first call) and shows a non-modal popup window (`Stage`) containing a read-only `TextArea` for displaying eKYC response data. On subsequent calls, reuses the existing window. The window is always brought to the top (`setAlwaysOnTop(true)`) and populated with the provided `data` string.

---

#### `updateSendButton()`
**Signature:** `private void updateSendButton()`

Centralized state machine that enables or disables the `sendAuthRequest` and `requestOtp` buttons based on current UI state. Logic:

1. If `idValue` is empty → disable both buttons and return.
2. Enable `requestOtp` only if `otpAuthType` is checked.
3. If OTP auth is selected but `otpValue` is empty → disable `sendAuthRequest` and return.
4. If any bio auth type is selected but `capture` is null (no capture performed) → disable `sendAuthRequest` and return.
5. Enable `sendAuthRequest` if at least one of demo, bio, or OTP auth is selected.

---

#### `updateBioPane()`
**Signature:** `private void updateBioPane()`

Enables or disables `bioAnchorPane` and the individual `fingerCount`/`irisCount` dropdowns based on which biometric check boxes are currently selected.

---

#### `onFingerPrintAuth()`, `onIrisAuth()`, `onFaceAuth()`
**Signature:** `private void onFingerPrintAuth/onIrisAuth/onFaceAuth()`  
**Annotation:** `@FXML` (triggered when the corresponding check box is toggled)

All three methods delegate to `updateBioCapture()`, which resets `capture` and `previousHash` to `null` (forcing a new capture for the newly selected combination), then calls `updateBioPane()` and `updateSendButton()`.

---

#### `onOTPAuth()`
**Signature:** `private void onOTPAuth()`  
**Annotation:** `@FXML`

Clears `responsetextField`, toggles the `otpAnchorPane` enabled/disabled state, and calls `updateSendButton()`.

---

#### `onIdTypeChange()`, `onSubTypeSelection()`
**Signature:** `private void onIdTypeChange/onSubTypeSelection()`  
**Annotation:** `@FXML`

Both methods simply clear `responsetextField` when the user changes the ID type or sub-type selection.

---

#### `onDemoAuth()`
**Signature:** `private void onDemoAuth()`  
**Annotation:** `@FXML`

Enables or disables `demoInputData` based on whether `demoAuthType` is checked, then calls `updateSendButton()`.

---

#### `onEKyc()`
**Signature:** `private void onEKyc()`  
**Annotation:** `@FXML`

Shows or hides the `ekycResponse` button based on whether eKYC mode is selected and `ekycResponseText` is non-blank.

---

#### `onReset()`
**Signature:** `private void onReset()`  
**Annotation:** `@FXML`

Displays a JavaFX confirmation `Alert` ("Are you sure to reset?"). If the user confirms (clicks "Yes"), delegates to `reset()`.

---

#### `reset()`
**Signature:** `private void reset()`

Resets the entire form to its initial empty state:
- Resets all combo boxes to their default selections.
- Clears `idValue` and `otpValue`.
- Unchecks all authentication-type check boxes.
- Resets `idTypebox` to `"UIN"`.
- Disables `otpAnchorPane`, `bioAnchorPane`, `demoInputData`.
- Disables `requestOtp`, enables `sendAuthRequest`.
- Clears `capture`, `previousHash`, `ekycResponseText`.
- Hides `ekycResponse` button.
- Calls `updateBioPane()` and `updateSendButton()` to re-synchronize the final state.

---

#### `onOtpValueUpdate()`
**Signature:** `private void onOtpValueUpdate()`  
**Annotation:** `@FXML`

Called when the OTP text field changes. Delegates to `updateSendButton()`.

---

#### `viewKycResponse()`
**Signature:** `private void viewKycResponse()`  
**Annotation:** `@FXML`

Reserved FXML handler (currently empty body). Intended for a future "View KYC Response" action.

---

### 4.4 Biometric Capture Functions

#### `onCapture()`
**Signature:** `private void onCapture() throws Exception`  
**Annotation:** `@FXML` (triggered by the "Capture" button)

Orchestrates biometric capture for all selected modalities in sequence:

1. Resets `previousHash` to `null`.
2. If `faceAuthType` is selected, calls `captureFace()`. If the result does not contain `"biometrics"`, aborts and calls `updateSendButton()`.
3. If `fingerAuthType` is selected, calls `captureFingerprint()`. Same abort logic.
4. If `irisAuthType` is selected, calls `captureIris()`. Same abort logic.
5. Passes all successful capture results to `combineCaptures(bioCaptures)` and stores the combined JSON in `capture`.
6. Calls `updateSendButton()` to enable the "Send Auth Request" button if conditions are met.

---

#### `captureFingerprint()`
**Signature:** `private String captureFingerprint() throws Exception`

Builds a biometric capture request for fingerprint:
1. Updates `responsetextField` to display `"Capturing Fingerprint..."`.
2. Retrieves the capture request template via `getCaptureRequestTemplate()`.
3. Substitutes all placeholders (`$timeout`, `$count`, `$deviceId`, `$domainUri`, `$deviceSubId`, `$captureTime`, `$previousHash`, `$requestedScore`, `$type`, `$bioSubType`, `$name`, `$value`, `$env`) with values from the Spring `Environment`.
4. Calls `capturebiometrics(requestBody)` and returns the SBI JSON response.

---

#### `captureIris()`
**Signature:** `private String captureIris() throws Exception`

Same structure as `captureFingerprint()` but uses iris-specific property keys (`ida.request.captureIris.*`) and `getIrisDeviceSubId()`.

---

#### `captureFace()`
**Signature:** `private String captureFace() throws Exception`

Same structure but uses face-specific property keys (`ida.request.captureFace.*`) and `getFaceDeviceSubId()`. Face count is always `"1"`.

---

#### `capturebiometrics(String requestBody)`
**Signature:** `private String capturebiometrics(String requestBody) throws Exception`

Low-level function that sends the HTTP CAPTURE request to the local SBI device service:

1. Creates an Apache `CloseableHttpClient`.
2. Builds an HTTP request using the non-standard `CAPTURE` method (`RequestBuilder.create("CAPTURE")`) with the JSON body, targeting `ida.captureRequest.uri` (default: `http://127.0.0.1:4501/capture`).
3. Reads the full response body.
4. Parses the JSON response: if `"biometrics"` key is absent, displays a red error message.
5. Iterates over each biometric segment. For each:
   - Reads the `error.errorCode` field. `"0"` or `"100"` means success.
   - On success: decodes the `data` JWS token (splits on `.`, Base64-decodes the payload), logs bio type and sub-type, and stores the `hash` field in `previousHash`.
   - On failure: displays a red "Capture Failed" message and breaks out of the loop.
6. Returns the raw JSON response string for use by `combineCaptures()`.

---

#### `combineCaptures(List<String> bioCaptures)`
**Signature:** `private String combineCaptures(List<String> bioCaptures)`

Merges biometric capture results from multiple modalities into a single JSON object:

- Filters out null/invalid captures.
- If only one valid capture, returns it as-is.
- If multiple captures: parses each JSON string as a `Map`, extracts the `"biometrics"` list from each, merges all lists into a single flat list, and serializes the combined result as `{"biometrics": [...]}`.

---

#### `getCaptureRequestTemplate()`
**Signature:** `private String getCaptureRequestTemplate()`

Reads the Base64-encoded capture request template from the `ida.request.captureRequest.template` property and decodes it to a UTF-8 string using `cryptoUtil.decodeBase64(...)`.

---

#### `getBioSubType(String count, String bioValue)`
**Signature:** `private String getBioSubType(String count, String bioValue)`

Given a count and a bio sub-type value string, generates a comma-separated list of quoted sub-type values for insertion into the capture request template. For example, count `"3"` with value `"UNKNOWN"` produces `"UNKNOWN","UNKNOWN","UNKNOWN"`.

---

#### `getFingerCount()`
Returns the selected finger count from the `fingerCount` combo box (defaults to `"1"` if null).

#### `getIrisCount()`
Returns the selected iris count as a 1-based index string (1=Left, 2=Right, 3=Both).

#### `getFaceCount()`
Always returns `"1"` (face capture is always single).

#### `getFingerDeviceSubId()`
Returns the `finger.device.subid` property value (defaults to `"0"`).

#### `getIrisDeviceSubId()`
Returns the `iris.device.subid` property value. If absent, derives the sub-ID from the iris count selection (1=Left, 2=Right, 3=Both).

#### `getFaceDeviceSubId()`
Returns the `face.device.subid` property value (defaults to `"0"`).

#### `getPreviousHash()`
Returns `previousHash` if non-null, or an empty string `""` otherwise.

#### `getCaptureTime()`
Returns the current UTC timestamp formatted as `"yyyy-MM-dd'T'HH:mm:ss'Z'"`.

---

### 4.5 OTP Functions

#### `onRequestOtp()`
**Signature:** `private void onRequestOtp()`  
**Annotation:** `@FXML`

Constructs and sends an OTP request to the MOSIP IDA OTP endpoint:

1. Builds an `OtpRequestDTO` with:
   - `id` = `"mosip.identity.otp"`
   - `individualId` = value from `idValue` text field
   - `individualIdType` = selected value from `idTypebox`
   - `otpChannel` = `["email"]` (single-element list)
   - `requestTime` = current UTC timestamp
   - `transactionID` = `"1234567890"` (static, from `getTransactionID()`)
   - `version` = `"1.0"`
2. Serializes to JSON using Jackson.
3. Creates an SSL-disabled `RestTemplate` with the auth-token interceptor via `createTemplate()`.
4. Adds a `signature` header (JWS of the JSON body) and `Content-type: application/json`.
5. POSTs to `ida.otp.url`.
6. On a 2xx response: checks for errors in the response body. Displays `"OTP Request Success"` (green) or `"OTP Request Failed"` (red).
7. On non-2xx: displays `"OTP Request Failed with Error"` (red).

---

### 4.6 Authentication Request Functions

#### `onSendAuthRequest()`
**Signature:** `private void onSendAuthRequest() throws Exception`  
**Annotation:** `@FXML`

The primary authentication orchestration function. Steps:

1. **Build `AuthRequestDTO`:** Sets `requestedAuth` (auth type flags), `individualId`, `individualIdType`, `env`, `domainUri`.
2. **Build request block (`RequestDTO`):** Sets timestamp. If OTP auth is enabled, sets `otp`.
3. **Add biometrics:** If bio auth is selected, reads `capture` and adds the `"biometrics"` array to the identity block.
4. **Add demographics:** If demo auth is selected, reads `demoInputData` text (defaulting to `{}`) and adds it as `"demographics"`.
5. **Encrypt:** Calls `kernelEncrypt(encryptionRequestDto, false)` to get:
   - `encryptedIdentity` – AES-GCM encrypted identity block (Base64)
   - `encryptedSessionKey` – RSA-OAEP encrypted AES key (Base64)
   - `requestHMAC` – HMAC of the identity block, AES-encrypted (Base64)
   - `thumbprint` – SHA-256 fingerprint of the IDA certificate (Base64)
6. **Finalize `AuthRequestDTO`:** Sets `transactionID`, `requestTime`, `consentObtained=true`, `id` (auth or eKYC), `version`, `thumbprint`. Replaces `request`, `requestSessionKey`, `requestHMAC` with encrypted values.
7. **Sign and POST:** Serializes to JSON, adds `signature` header (JWS), POSTs to `ida.auth.url` or `ida.ekyc.url`.
8. **Handle response:** On success, checks `authStatus` or `kycStatus`. Displays green/red message. If eKYC success, calls `decryptEkycResponse()` and shows the data in a dialog.

---

#### `getAuthRequestId()`
**Signature:** `private String getAuthRequestId()`

Returns `"mosip.identity.kyc"` if eKYC mode is selected, else `"mosip.identity.auth"` (both configurable via `authRequestId` property).

---

#### `getUrl()`
**Signature:** `private String getUrl()`

Returns `ida.ekyc.url` in eKYC mode, else `ida.auth.url`.

---

#### `isBioAuthType()`
Returns `true` if any of `fingerAuthType`, `irisAuthType`, or `faceAuthType` is checked.

#### `isOtpAuthType()`
Returns `true` if `otpAuthType` is checked.

#### `isDemoAuthType()`
Returns `true` if `demoAuthType` is checked. Also calls `updateSendButton()` as a side-effect.

#### `isEkycAuthType()`
Returns `true` if `eKyc` is checked.

---

### 4.7 Encryption Functions

#### `kernelEncrypt(EncryptionRequestDto, boolean isInternal)`
**Signature:** `private EncryptionResponseDto kernelEncrypt(EncryptionRequestDto encryptionRequestDto, boolean isInternal) throws Exception`

Performs hybrid AES+RSA encryption of the identity block:

1. **Serialize** the identity request map to JSON string.
2. **Generate AES-256 session key** using `cryptoUtil.genSecKey()`.
3. **Symmetric-encrypt** the JSON identity block using the AES session key → `encryptedIdentity` (Base64 URL-safe).
4. **Fetch IDA certificate** from the MOSIP server via `getCertificate(...)`.
5. **Asymmetric-encrypt** the AES session key bytes using the IDA public key (RSA-OAEP) → `encryptedSessionKey` (Base64 URL-safe).
6. **Compute HMAC**: Computes SHA-256 digest of the identity JSON, then symmetrically encrypts it with the AES key → `requestHMAC` (Base64 URL-safe).
7. **Compute thumbprint**: SHA-256 of the DER-encoded certificate → `thumbprint` (Base64).
8. Returns an `EncryptionResponseDto` containing all four values.

---

#### `getCertificateThumbprint(Certificate cert)`
**Signature:** `private byte[] getCertificateThumbprint(Certificate cert) throws CertificateEncodingException`

Computes and returns `SHA-256(cert.getEncoded())` using Apache Commons `DigestUtils`.

---

#### `getCertificate(String data, boolean isInternal)`
**Signature:** `public X509Certificate getCertificate(String data, boolean isInternal) throws ...`

Fetches the IDA public certificate from the MOSIP keymanager service:

1. Creates a `CryptomanagerRequestDto` with application ID `"IDA"`, reference ID from `ida.reference.id`, and a UTC timestamp.
2. Calls the `ida.certificate.url` endpoint (GET with query parameters `applicationId` and `referenceId`).
3. Extracts the PEM-encoded certificate from the response, strips PEM headers, Base64-decodes, and parses it as an `X509Certificate` using `CertificateFactory`.

---

#### `trimBeginEnd(String pKey)` *(static)*
**Signature:** `public static String trimBeginEnd(String pKey)`

Strips PEM armor (`-----BEGIN ... -----` / `-----END ... -----`) and all whitespace from a PEM-encoded string, returning the raw Base64 content. Used before decoding certificates received from the server.

---

### 4.8 eKYC Functions

#### `decryptEkycResponse(ResponseEntity<Map> authResponse)`
**Signature:** `private void decryptEkycResponse(ResponseEntity<Map> authResponse) throws Exception`

Decrypts the encrypted KYC payload returned by the MOSIP eKYC endpoint:

1. Retrieves the EKYC partner private key via `keyMgrUtil.getKeyEntry(...)` with partner ID `"ekyc"`.
2. Extracts `identity` and `sessionKey` from the response body.
3. **If `sessionKey` is null** (old format): splits the `identity` string at the `#KEY_SPLITTER#` separator to obtain the encrypted session key and encrypted data portions using `splitEncryptedData(...)`.
4. **If `sessionKey` is present** (new format): the session key and encrypted data are already separated.
5. **Decrypts the AES session key** using the EKYC private key with RSA-OAEP/SHA-256 (`decryptSecretKey()`).
6. **Decrypts the KYC data** using AES-GCM:
   - Extracts the 16-byte GCM nonce from the last 16 bytes of the encrypted KYC data.
   - Initializes AES cipher in `AES/GCM/PKCS5Padding` with the nonce and decrypted session key.
   - Decrypts the remaining bytes to get the plaintext KYC JSON.
7. Pretty-prints the result using Jackson and stores it in `ekycResponseText`.

---

#### `splitEncryptedData(String data)`
**Signature:** `public Map<String, String> splitEncryptedData(String data)`

Decodes a Base64-encoded combined data block and splits it at the first occurrence of the `mosip.kernel.data-key-splitter` separator (default `#KEY_SPLITTER#`). Returns a `Map` with keys `"encryptedSessionKey"` and `"encryptedData"` (both Base64-encoded).

---

#### `splitAtFirstOccurance(byte[] strBytes, byte[] sepBytes)` *(static)*
**Signature:** `private static byte[][] splitAtFirstOccurance(byte[] strBytes, byte[] sepBytes)`

Splits a byte array at the **first** occurrence of a separator byte sequence. Returns a 2-element array: bytes before the separator, and bytes after it. If the separator is not found, returns the original array and an empty array.

---

#### `findIndex(byte[] arr, byte[] subarr)` *(static)*
**Signature:** `private static int findIndex(byte[] arr, byte[] subarr)`

Finds the index of the first occurrence of `subarr` within `arr` using `IntStream.range`. Returns `-1` if not found.

---

#### `decryptSecretKey(PrivateKey privKey, byte[] encKey)`
**Signature:** `private byte[] decryptSecretKey(PrivateKey privKey, byte[] encKey) throws ...`

Decrypts an RSA-encrypted AES session key using the EKYC partner's private key. Uses `RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING` cipher with `OAEPParameterSpec(SHA-256, MGF1, MGF1ParameterSpec.SHA256)` to match the server-side encryption scheme.

---

### 4.9 Certificate and Key Utilities

These functions are used internally by the auth and eKYC flows.

#### `getSignature(String reqJson)`
**Signature:** `private String getSignature(String reqJson) throws ...`

Convenience wrapper that calls `sign(reqJson, false)`.

---

#### `sign(String data, boolean isPayloadRequired)`
**Signature:** `public String sign(String data, boolean isPayloadRequired) throws ...`

Signs the given data by calling `signatureUtil.sign(data, false, true, false, null, getKeysDirPath(), env.getProperty("partnerId"))`. The parameters specify: no payload embedding, include certificate, no certificate hash, no URL, partner key from the `keys/` directory, and the configured partner ID.

---

### 4.10 HTTP and SSL Helpers

#### `createTemplate()`
**Signature:** `private RestTemplate createTemplate() throws KeyManagementException, NoSuchAlgorithmException`

Creates a Spring `RestTemplate` configured with:
1. SSL checking disabled via `turnOffSslChecking()`.
2. A `ClientHttpRequestInterceptor` that, before each request, calls `generateAuthToken()` and injects the resulting token as both a `Cookie: Authorization=<token>` header and an `Authorization: Authorization=<token>` header.

---

#### `generateAuthToken()`
**Signature:** `private String generateAuthToken()`

Obtains an authentication token from the MOSIP auth manager service (`ida.authmanager.url`) using the configured `clientId`, `secretKey`, and `appId`. Sends a POST request via Spring WebFlux `WebClient`, extracts the `Authorization` cookie from the response, and returns its value. Returns an empty string if no cookie is present.

---

#### `turnOffSslChecking()` *(static)*
**Signature:** `public static void turnOffSslChecking() throws KeyManagementException, NoSuchAlgorithmException`

Installs an all-trusting `X509TrustManager` (`UNQUESTIONING_TRUST_MANAGER`) into the default `SSLContext`, disabling SSL certificate validation for all HTTPS connections made via `HttpsURLConnection`. This is appropriate only in test/demo environments.

> **Warning:** This disables all SSL certificate validation. Do not use in production.

---

#### `getHeaders(CryptomanagerRequestDto req)` *(unused)*
**Signature:** `private HttpEntity<CryptomanagerRequestDto> getHeaders(CryptomanagerRequestDto req)`

Creates an `HttpEntity` with `Accept: application/json` header. Annotated `@SuppressWarnings("unused")`.

---

### 4.11 Miscellaneous Helpers

#### `getUTCCurrentDateTimeISOString()` *(static)*
**Signature:** `public static String getUTCCurrentDateTimeISOString()`

Returns the current UTC date-time as an ISO-8601 formatted string using `DateUtils.formatToISOString(DateUtils.getUTCCurrentDateTime())` from the MOSIP kernel utility.

---

#### `getTransactionID()` *(static)*
**Signature:** `public static String getTransactionID()`

Returns the static string `"1234567890"` as the transaction ID. This is a placeholder for demo purposes.

---

## 5. Cryptography Helper – `CryptoUtility`

**Package:** `io.mosip.authentication.demo.helper`  
**Annotation:** `@Component`

`CryptoUtility` is a Spring-managed wrapper around the MOSIP kernel `CryptoCoreSpec` interface, which provides the underlying AES-GCM symmetric and RSA-OAEP asymmetric cryptographic operations. It also registers the BouncyCastle security provider.

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `SYM_ALGORITHM` | `"AES"` | Algorithm name used for symmetric key generation. |
| `SYM_ALGORITHM_LENGTH` | `256` | AES key size in bits. |

### Fields

| Field | Description |
|-------|-------------|
| `bouncyCastleProvider` | Static `BouncyCastleProvider` instance registered with `java.security.Security`. |
| `cryptoCore` | Autowired `CryptoCoreSpec<byte[],byte[],SecretKey,PublicKey,PrivateKey,String>` from the MOSIP kernel. |

### Static Initializer

Calls `addProvider()` to register BouncyCastle as a JCE security provider on class load.

---

### Methods

#### `symmetricEncrypt(byte[] data, SecretKey secretKey)`
**Signature:** `public byte[] symmetricEncrypt(byte[] data, SecretKey secretKey) throws ...`

Encrypts `data` using the provided AES `SecretKey` via `cryptoCore.symmetricEncrypt(secretKey, data, null)`. The kernel implementation uses AES-GCM with PKCS5Padding. Returns the ciphertext byte array.

---

#### `symmetricDecrypt(SecretKey secretKey, byte[] encryptedDataByteArr)`
**Signature:** `public byte[] symmetricDecrypt(SecretKey secretKey, byte[] encryptedDataByteArr) throws ...`

Decrypts `encryptedDataByteArr` using the provided AES `SecretKey` via `cryptoCore.symmetricDecrypt(secretKey, encryptedDataByteArr, null)`. Returns the plaintext byte array.

---

#### `addProvider()` *(static, private)*
**Signature:** `private static BouncyCastleProvider addProvider()`

Creates a new `BouncyCastleProvider` and registers it with `Security.addProvider(...)`. Returns the provider instance for assignment to `bouncyCastleProvider`.

---

#### `genSecKey()`
**Signature:** `public SecretKey genSecKey() throws NoSuchAlgorithmException`

Generates a new AES-256 `SecretKey`:
1. Obtains a `KeyGenerator` instance for `"AES"` using the BouncyCastle provider.
2. Initializes it with a key size of 256 bits and a `SecureRandom` instance.
3. Returns the generated key.

---

#### `asymmetricEncrypt(byte[] data, PublicKey publicKey)`
**Signature:** `public byte[] asymmetricEncrypt(byte[] data, PublicKey publicKey) throws ...`

Encrypts `data` using the provided RSA `PublicKey` via `cryptoCore.asymmetricEncrypt(publicKey, data)`. The kernel uses RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING. Returns the encrypted byte array.

---

#### `decodeBase64(String data)`
**Signature:** `public byte[] decodeBase64(String data)`

Decodes a Base64-encoded string using Apache Commons `Base64.decodeBase64(data)`. Returns the decoded byte array. Supports both standard and URL-safe Base64.

---

## 6. Key Management Utility – `KeyMgrUtil`

**Package:** `io.mosip.authentication.demo.service.util`  
**Annotation:** `@Component`

`KeyMgrUtil` manages all PKI operations: generating RSA key pairs, building X.509 certificate chains (CA → Intermediate → Partner), loading/storing PKCS12 key stores, and reading/writing certificate files.

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DOMAIN_URL` | `"mosip.base.url"` | Property key for the MOSIP base URL (used to derive the keys directory name). |
| `CERTIFICATE_TYPE` | `"X.509"` | Certificate type string for `CertificateFactory`. |
| `CA_P12_FILE_NAME` | `"-ca.p12"` | Suffix for the CA private key file. |
| `INTER_P12_FILE_NAME` | `"-inter.p12"` | Suffix for the Intermediate CA key file. |
| `PARTNER_P12_FILE_NAME` | `"-partner.p12"` | Suffix for the Partner private key file. |
| `CA_CER_FILE_NAME` | `"-ca.cer"` | Suffix for the CA certificate file. |
| `INTER_CER_FILE_NAME` | `"-inter.cer"` | Suffix for the Intermediate certificate file. |
| `PARTNER_CER_FILE_NAME` | `"-partner.cer"` | Suffix for the Partner certificate file. |
| `TEMP_P12_PWD` | `"qwerty@123"` | Default PKCS12 password (overridable via `p12.password` property). |
| `KEY_ALIAS` | `"keyalias"` | Alias used for all key store entries. |
| `KEY_STORE` | `"PKCS12"` | Key store type. |
| `RSA_ALGO` | `"RSA"` | Algorithm name for key pair generation. |
| `RSA_KEY_SIZE` | `2048` | RSA key size in bits. |
| `SIGN_ALGO` | `"SHA256withRSA"` | Algorithm used for signing certificates. |

---

### Public Methods

#### `convertToCertificate(String certData)`
**Signature:** `public Certificate convertToCertificate(String certData) throws IOException, CertificateException`

Parses a PEM-encoded certificate string into a Java `Certificate` object:
1. Wraps the string in a `PemReader` to extract the raw DER bytes.
2. Uses `CertificateFactory.getInstance("X.509")` to parse the DER bytes.
3. Returns the resulting `Certificate`.

---

#### `getPartnerCertificates(String partnerId, String organization, String dirPath)`
**Signature:** `public void getPartnerCertificates(String partnerId, String organization, String dirPath) throws ...`

Generates a three-tier PKI certificate chain for the given partner:

1. **CA level:** Loads or generates a self-signed CA key pair (`<partnerId>-ca.p12`). The CA has `KeyUsage.digitalSignature | KeyUsage.keyCertSign`. Certificate is valid for 1 year from now.
2. **Intermediate level:** Loads or generates an intermediate key pair (`<partnerId>-inter.p12`) signed by the CA private key.
3. **Partner level:** Loads or generates a partner key pair (`<partnerId>-partner.p12`) signed by the intermediate private key. For the `"EKYC"` partner ID, uses `keyEncipherment | encipherOnly | decipherOnly` key usage flags.
4. Writes PEM certificate files for each level (`-ca.cer`, `-inter.cer`, `-partner.cer`) via `writeContentToFile()`.

---

#### `getKeyEntry(String dirPath, String parterId)`
**Signature:** `public PrivateKeyEntry getKeyEntry(String dirPath, String parterId) throws ...`

Loads and returns the partner `PrivateKeyEntry` from the PKCS12 file `<dirPath>/<parterId>-partner.p12`. Returns `null` if the file does not exist.

---

#### `updatePartnerCertificate(String partnerType, X509Certificate updateCert, String dirPath)`
**Signature:** `public boolean updatePartnerCertificate(String partnerType, X509Certificate updateCert, String dirPath) throws ...`

Updates the partner certificate in the PKCS12 key store with a new certificate (e.g., one signed by MOSIP PMS):
1. Loads the existing partner PKCS12 file.
2. Verifies that the public key in `updateCert` matches the stored public key (prevents key mismatch).
3. Creates a new `PrivateKeyEntry` with the existing private key but the new certificate.
4. Stores it back to the PKCS12 file.
5. Returns `true` on success, `false` if no existing key file was found.

---

#### `getKeysDirPath()`
**Signature:** `public String getKeysDirPath()`

Derives the keys directory path from the `mosip.base.url` property by stripping `https://`, `http://`, and trailing slashes, then prepending the JVM temp directory: `<java.io.tmpdir>/IDA-<domain>`.

---

#### `getCertificateEntry(String dirPath, String partnerId)`
**Signature:** `public X509Certificate getCertificateEntry(String dirPath, String partnerId) throws ...`

Reads the partner PEM certificate file (`<partnerId>-partner.cer`) from `dirPath`, strips PEM headers using `trimBeginEnd()`, Base64-decodes the content, and parses it as an `X509Certificate`. Returns `null` if the file does not exist.

---

#### `trimBeginEnd(String pKey)` *(static)*
**Signature:** `public static String trimBeginEnd(String pKey)`

Identical to `IdaController.trimBeginEnd()`. Removes PEM `-----BEGIN ... -----` / `-----END ... -----` markers and all whitespace from the input string, leaving only the raw Base64 data.

---

### Private Methods

#### `writeContentToFile(String content, String filePath)`
**Signature:** `private void writeContentToFile(String content, String filePath) throws IOException`

Writes the string content to a new file at `filePath` using `Files.write(..., StandardOpenOption.CREATE_NEW)`. Throws an exception if the file already exists.

---

#### `getPrivateKeyEntry(String filePath)`
**Signature:** `private PrivateKeyEntry getPrivateKeyEntry(String filePath) throws ...`

Loads a `PrivateKeyEntry` from a PKCS12 file:
1. Checks if the file exists via `Files.exists(path)`.
2. Opens a `KeyStore` of type `PKCS12`, loads it with the P12 password.
3. Returns the entry under alias `"keyalias"`.
4. Returns `null` if the file does not exist.

---

#### `getCertificate(PrivateKeyEntry keyEntry)`
**Signature:** `private String getCertificate(PrivateKeyEntry keyEntry) throws IOException`

Serializes the certificate in `keyEntry` to PEM format using BouncyCastle's `JcaPEMWriter`. Returns the PEM string.

---

#### `generateKeys(PrivateKey signKey, String signCertType, String certType, String p12FilePath, KeyUsage keyUsage, LocalDateTime dateTime, LocalDateTime dateTimeExp, String organization)`
**Signature:** `private PrivateKeyEntry generateKeys(...) throws ...`

Generates a new RSA key pair and a signed X.509 certificate, then saves everything to a PKCS12 file:
1. Generates a 2048-bit RSA key pair using `KeyPairGenerator`.
2. Generates an X.509 certificate (self-signed if `signKey` is null, otherwise signed by `signKey`) via `generateX509Certificate(...)`.
3. Creates a `PrivateKeyEntry` with the new private key and certificate chain.
4. Stores it in a new PKCS12 key store at `p12FilePath`.
5. Creates parent directories if they don't exist.
6. Returns the `PrivateKeyEntry`.

---

#### `generateX509Certificate(PrivateKey signPrivateKey, PublicKey publicKey, String signCertType, String certType, KeyUsage keyUsage, LocalDateTime dateTime, LocalDateTime dateTimeExp, String organization)`
**Signature:** `private X509Certificate generateX509Certificate(...) throws ...`

Builds and returns an X.509 v3 certificate using BouncyCastle:
1. Constructs the issuer (`signCertType`) and subject (`certType`) `X500Name` objects via `getCertificateAttributes()`.
2. Sets validity from `dateTime` to `dateTimeExp`.
3. Generates a random serial number.
4. Signs the certificate with `signPrivateKey` using `SHA256withRSA`.
5. Adds extensions: `BasicConstraints(true)` (marks it as a CA cert), Subject Key Identifier, and the provided `KeyUsage`.

---

#### `getCertificateAttributes(String cn, String organization)` *(static)*
**Signature:** `private static X500Name getCertificateAttributes(String cn, String organization)`

Builds an `X500Name` with the following fixed/parameterized attributes:
- `C` = `"IN"` (Country: India)
- `ST` = `"KA"` (State: Karnataka)
- `O` = `organization`
- `OU` = `"IDA-TEST-ORG-UNIT"`
- `CN` = `cn`

---

#### `getP12Pass()`
**Signature:** `private char[] getP12Pass()`

Returns the PKCS12 password as a `char[]`. Reads from the `p12.password` property; falls back to `TEMP_P12_PWD` (`"qwerty@123"`) if the property is not set.

---

## 7. Signature Utility – `SignatureUtil`

**Package:** `io.mosip.authentication.demo.service.util`  
**Annotation:** `@Component`

`SignatureUtil` creates JWS (JSON Web Signature) tokens using the RS256 algorithm (RSA + SHA-256) to sign all outgoing IDA requests. The signed token is included in the `signature` HTTP header.

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `SIGN_ALGO` | `"RS256"` | JWS algorithm identifier. |

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `keyMgrUtil` | `KeyMgrUtil` | Used to load the partner private key and certificate. |

---

### Methods

#### `sign(String dataToSign, boolean includePayload, boolean includeCertificate, boolean includeCertHash, String certificateUrl, String dirPath, String partnerId)`
**Signature:** `public String sign(...) throws JoseException, NoSuchAlgorithmException, UnrecoverableEntryException, KeyStoreException, CertificateException, IOException, OperatorCreationException`

Creates a JWS (compact serialization) of `dataToSign`:

1. Creates a `JsonWebSignature` object from the jose4j library.
2. Loads the partner `PrivateKeyEntry` via `keyMgrUtil.getKeyEntry(dirPath, partnerId)`. Throws `KeyStoreException` if the key is not found.
3. Reads the `PrivateKey` for signing.
4. Retrieves the partner `X509Certificate` via `keyMgrUtil.getCertificateEntry(...)`. If not found, falls back to the certificate inside the `PrivateKeyEntry`.
5. **If `includeCertificate` is true:** adds the certificate to the JWS `x5c` header.
6. **If `includeCertHash` is true:** adds the SHA-256 certificate thumbprint to the `x5t#S256` header.
7. **If `certificateUrl` is non-null:** adds it as the `x5u` header.
8. Sets the payload to `dataToSign`.
9. Sets the algorithm to `"RS256"`.
10. Sets the private key and disables key validation (to allow non-standard keys).
11. **If `includePayload` is true:** returns the full compact serialization (`header.payload.signature`).
12. **If `includePayload` is false:** returns the detached-content compact serialization (`header..signature`), where the payload is omitted from the token.

In practice, `IdaController.sign()` calls this with `includePayload=false` and `includeCertificate=true`, producing a detached JWS where the receiver can independently verify the signature against the request body.

---

## 8. MJPEG Stream Runner – `MjpegTestRunner`

**Package:** `io.mosip.authentication.demo.service`  
**Implements:** `java.lang.Runnable`

`MjpegTestRunner` is a background thread that connects to an MJPEG stream URL (e.g., from a biometric device) and continuously reads JPEG frames to display in a JavaFX `ImageView`. This class is present for demonstration/debugging of device streams and is not part of the core authentication flow.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `CONTENT_LENGTH` | `String` (constant) | HTTP header prefix `"Content-Length:"` used during frame parsing. |
| `CONTENT_TYPE` | `String` (constant) | Expected MIME type `"Content-type: image/jpeg"`. |
| `url` | `URL` | The MJPEG stream URL. |
| `viewer` | `ImageView` | The JavaFX image view where frames are displayed. |
| `urlStream` | `InputStream` | The HTTP response input stream of the MJPEG connection. |
| `isRunning` | `boolean` | Loop control flag (volatile via `synchronized` methods). |

---

### Constructor: `MjpegTestRunner(ImageView viewer, URL url)`
**Throws:** `IOException`

Stores `viewer` and `url`, then immediately calls `start()`. *(Note: The `isRunning` field is initialized to `true` and `start()` is called in the constructor; the TODO comments indicate this behavior is subject to change.)*

---

### Methods

#### `start()`
**Signature:** `private void start() throws IOException`

Establishes an HTTP POST connection to `url`:
1. Opens an `HttpURLConnection` and sets the method to `"POST"`.
2. Sends a hardcoded SBI capture request body (JSON for a Fingerprint capture).
3. Sets a 5-second read timeout.
4. Connects and stores the response input stream in `urlStream`.

---

#### `stop()`
**Signature:** `public synchronized void stop()`

Sets `isRunning = false` to signal the `run()` loop to exit on the next iteration.

---

#### `run()`
**Signature:** `public void run()`

Main loop (executed on a background thread):
1. While `isRunning` is `true`:
   a. Calls `retrieveNextImage()` to read the next JPEG frame as a byte array.
   b. Wraps it in a `ByteArrayInputStream` and constructs a JavaFX `Image`.
   c. Updates `viewer` with the new image.
   d. On `SocketTimeoutException` or `IOException`, calls `stop()` to exit the loop.
2. After the loop exits, closes `urlStream`.

---

#### `addTimestampToFrame(BufferedImage frame)`
**Signature:** `private void addTimestampToFrame(BufferedImage frame)`

Draws the current date/time string as white text near the bottom of the given `BufferedImage` using `Graphics2D`. This method exists but is not currently called from `run()`.

---

#### `retrieveNextImage()`
**Signature:** `private byte[] retrieveNextImage() throws IOException`

Parses one JPEG frame from the MJPEG stream:
1. Reads bytes from `urlStream` character by character, accumulating the HTTP chunk header.
2. When the `"Content-Length:"` header is found, captures the following bytes until a newline to determine the frame size in bytes.
3. Skips bytes in the stream until the JPEG start-of-image marker (`0xFF`) is found.
4. Allocates a `byte[]` of size `contentLength + 1`, prefills `[0] = 0xFF`, and reads the rest of the frame.
5. Returns the complete JPEG byte array.

---

## 9. Data Transfer Objects (DTOs)

All DTOs use [Lombok](https://projectlombok.org/) annotations to auto-generate getters, setters, `equals`, `hashCode`, and `toString`. Most use `@Data`; some add `@NoArgsConstructor` and `@AllArgsConstructor`.

---

### 9.1 `BaseAuthRequestDTO`

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Request identifier (e.g., `"mosip.identity.auth"`). |
| `version` | `String` | API version string (e.g., `"1.0"`). |

Base class for all authentication request DTOs.

---

### 9.2 `AuthRequestDTO`

**Extends:** `BaseAuthRequestDTO`

The top-level request object sent to the IDA authentication endpoint.

| Field | Type | Description |
|-------|------|-------------|
| `requestedAuth` | `AuthTypeDTO` | Flags indicating which authentication types are being used. |
| `transactionID` | `String` | Unique transaction identifier for this request. |
| `requestTime` | `String` | ISO-8601 UTC timestamp of the request. |
| `request` | `RequestDTO` | The encrypted/assembled request payload block. |
| `consentObtained` | `boolean` | Must be `true`; indicates that the individual's consent was obtained. |
| `individualId` | `String` | The UIN, VID, or USERID of the individual being authenticated. |
| `individualIdType` | `String` | Type of `individualId`: `"UIN"`, `"VID"`, or `"USERID"`. |
| `requestHMAC` | `String` | Base64-encoded AES-GCM encrypted SHA-256 HMAC of the request block. |
| `thumbprint` | `String` | Base64-encoded SHA-256 fingerprint of the IDA certificate used for encryption. |
| `requestSessionKey` | `String` | Base64-encoded RSA-encrypted AES session key. |
| `env` | `String` | Deployment environment (e.g., `"Staging"`). |
| `domainUri` | `String` | Domain URI of the MOSIP environment. |

---

### 9.3 `AuthTypeDTO`

Flags used in `AuthRequestDTO.requestedAuth` to specify which authentication factors are present in this request.

| Field | Type | Description |
|-------|------|-------------|
| `demo` | `boolean` | `true` if demographic data is included. |
| `bio` | `boolean` | `true` if biometric data is included. |
| `otp` | `boolean` | `true` if an OTP is included. |
| `pin` | `boolean` | `true` if a PIN is included (not currently used in the UI). |

---

### 9.4 `RequestDTO`

Represents the unencrypted identity/request block before it is encrypted into `AuthRequestDTO.request`.

| Field | Type | Description |
|-------|------|-------------|
| `otp` | `String` | OTP value entered by the user (for OTP auth). |
| `timestamp` | `String` | ISO-8601 UTC timestamp of when the request block was created. |
| `demographics` | `IdentityDTO` | Demographic identity information (for demo auth). |
| `biometrics` | `List<BioIdentityInfoDTO>` | List of biometric identity segments (for bio auth). |

---

### 9.5 `OtpRequestDTO`

Sent to the IDA OTP endpoint to trigger OTP generation and delivery.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Request identifier: `"mosip.identity.otp"`. |
| `version` | `String` | API version (e.g., `"1.0"`). |
| `transactionID` | `String` | Unique transaction identifier. |
| `requestTime` | `String` | ISO-8601 UTC timestamp. |
| `individualId` | `String` | The UIN/VID/USERID. |
| `individualIdType` | `String` | Type of the individual ID. |
| `otpChannel` | `List<String>` | Delivery channels (e.g., `["email"]`). |

---

### 9.6 `IdentityDTO`

Holds the demographic identity data for demographic authentication.

| Field | Type | Description |
|-------|------|-------------|
| `age` | `String` | Age of the individual. |
| `dob` | `String` | Date of birth. |
| `name` | `List<IdentityInfoDTO>` | Multi-language name entries. |
| `gender` | `List<IdentityInfoDTO>` | Multi-language gender entries. |
| `phoneNumber` | `String` | Phone number. |
| `emailId` | `String` | Email address. |
| `addressLine1/2/3` | `List<IdentityInfoDTO>` | Multi-language address lines. |
| `location1/2/3` | `List<IdentityInfoDTO>` | Multi-language location fields. |
| `postalCode` | `String` | Postal/PIN code. |
| `fullAddress` | `List<IdentityInfoDTO>` | Multi-language full address. |

---

### 9.7 `IdentityInfoDTO`

A single multi-language identity attribute value.

| Field | Type | Description |
|-------|------|-------------|
| `language` | `String` | BCP-47 language code (e.g., `"eng"`, `"ara"`). |
| `value` | `String` | The attribute value in the specified language. |

---

### 9.8 `BioIdentityInfoDTO`

Represents a single biometric segment as returned by the SBI device service and as included in the authentication request.

| Field | Type | Description |
|-------|------|-------------|
| `data` | `DataDTO` | The biometric payload metadata (decoded from the JWS token). |
| `hash` | `String` | Hash of this biometric segment (used as `previousHash` for the next capture). |
| `sessionKey` | `String` | Session key for this biometric segment (encrypted). |
| `signature` | `String` | JWS signature of the biometric data. |

---

### 9.9 `DataDTO`

Contains the metadata extracted from the SBI biometric JWS payload.

| Field | Type | Description |
|-------|------|-------------|
| `bioType` | `String` | Biometric type: `"Finger"`, `"Iris"`, `"Face"`. |
| `bioSubType` | `String` | Sub-type: e.g., `"Left IndexFinger"`, `"Left"`, `"UNKNOWN"`. |
| `bioValue` | `String` | The actual biometric data (ISO/CBEFF encoded, Base64). |
| `deviceCode` | `String` | Unique device code. |
| `deviceProviderID` | `String` | Device provider identifier. |
| `deviceServiceID` | `String` | Device service identifier. |
| `deviceServiceVersion` | `String` | SBI version. |
| `transactionID` | `String` | Transaction ID used during capture. |
| `timestamp` | `String` | Capture timestamp. |
| `mosipProcess` | `String` | MOSIP process context (e.g., `"Auth"`). |
| `environment` | `String` | Deployment environment. |
| `version` | `String` | SBI spec version. |

---

### 9.10 `EncryptionRequestDto`

Input to the `kernelEncrypt()` function.

| Field | Type | Description |
|-------|------|-------------|
| `identityRequest` | `Map<String, Object>` | The full identity/request block (OTP, biometrics, demographics) to be encrypted. |

---

### 9.11 `EncryptionResponseDto`

Output from the `kernelEncrypt()` function. All fields are Base64 URL-safe encoded.

| Field | Type | Description |
|-------|------|-------------|
| `encryptedSessionKey` | `String` | RSA-OAEP encrypted AES session key. |
| `encryptedIdentity` | `String` | AES-GCM encrypted identity block. |
| `requestHMAC` | `String` | AES-GCM encrypted SHA-256 HMAC of the identity block. |
| `thumbprint` | `String` | SHA-256 thumbprint of the IDA certificate. |

---

### 9.12 `CryptomanagerRequestDto`

Used when calling the MOSIP keymanager service to fetch the IDA encryption certificate.

| Field | Type | Description |
|-------|------|-------------|
| `applicationId` | `String` | Application identifier (e.g., `"IDA"`). |
| `data` | `String` | Base64-encoded data to be processed. |
| `referenceId` | `String` | Reference ID for the key to use (e.g., `"PARTNER"`). |
| `salt` | `String` | Optional salt value. |
| `timeStamp` | `String` | ISO-8601 UTC timestamp. |

---

### 9.13 `CertificateChainResponseDto`

Holds the three PEM certificates generated by `KeyMgrUtil.getPartnerCertificates()`.

| Field | Type | Description |
|-------|------|-------------|
| `caCertificate` | `String` | PEM-encoded CA certificate. |
| `interCertificate` | `String` | PEM-encoded Intermediate CA certificate. |
| `partnerCertificate` | `String` | PEM-encoded Partner certificate. |

---

### 9.14 `JWTSignatureRequestDto`

Data model for requesting a JWT signature (not currently used in the main flow; present for potential API use).

| Field | Type | Description |
|-------|------|-------------|
| `dataToSign` | `String` | Base64-encoded JSON data to sign. Required. |
| `applicationId` | `String` | Application ID to use for signing. |
| `referenceId` | `String` | Reference ID of the signing key. |
| `includePayload` | `Boolean` | Include payload in JWS header. |
| `includeCertificate` | `Boolean` | Include certificate in JWS header. |
| `includeCertHash` | `Boolean` | Include certificate SHA-256 hash in JWS header. |
| `certificateUrl` | `String` | URL for the certificate (`x5u` header). |

---

### 9.15 `JWTSignatureResponseDto`

Response model for a JWT signature operation.

| Field | Type | Description |
|-------|------|-------------|
| `jwtSignedData` | `String` | The compact JWS serialization. |
| `timestamp` | `LocalDateTime` | The time at which the signing occurred. |

---

## 10. Configuration and Properties

The application is configured via `src/main/resources/application.properties`. All properties can be overridden using JVM system properties (`-Dproperty=value`).

### Cryptography Properties

| Property | Description |
|----------|-------------|
| `mosip.kernel.crypto.asymmetric-algorithm-name` | RSA asymmetric cipher: `RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING` |
| `mosip.kernel.crypto.symmetric-algorithm-name` | AES symmetric cipher: `AES/GCM/PKCS5Padding` |
| `mosip.kernel.keygenerator.asymmetric-key-length` | RSA key size: `2048` |
| `mosip.kernel.keygenerator.symmetric-key-length` | AES key size: `256` |
| `mosip.kernel.data-key-splitter` | Separator between encrypted session key and encrypted data in combined payloads: `#KEY_SPLITTER#` |
| `mosip.kernel.crypto.gcm-tag-length` | GCM authentication tag length in bits: `128` |

### Partner / Authentication Properties

| Property | Description |
|----------|-------------|
| `partnerId` | Partner ID used for PKI and as path component in IDA URLs. |
| `partnerApiKey` | Partner API key used as path component in IDA URLs. |
| `partnerOrg` | Organization name for auto-generated certificates. Defaults to `partnerId`. |
| `mispLicenseKey` | MISP license key used as a path component in IDA URLs. |
| `clientId` | Client ID for MOSIP auth manager token generation. |
| `secretKey` | Secret key for MOSIP auth manager token generation. |
| `appId` | Application ID for MOSIP auth manager token generation. |
| `p12.password` | Password for PKCS12 key store files. Default: `qwerty@123`. |
| `mosip.sign.refid` | Reference ID for the signing key. Default: `SIGN`. |
| `authRequestId` | Overrides the auth request `id` field. Default: `mosip.identity.auth` / `mosip.identity.kyc`. |

### Endpoint URLs

| Property | Description |
|----------|-------------|
| `mosip.base.url` | Base URL of the MOSIP deployment (e.g., `https://qa.mosip.io`). |
| `ida.otp.url` | OTP request URL: `${mosip.base.url}/idauthentication/v1/otp/${mispLicenseKey}/${partnerId}/${partnerApiKey}` |
| `ida.auth.url` | Authentication URL: `${mosip.base.url}/idauthentication/v1/auth/${mispLicenseKey}/${partnerId}/${partnerApiKey}` |
| `ida.ekyc.url` | eKYC URL: `${mosip.base.url}/idauthentication/v1/kyc/${mispLicenseKey}/${partnerId}/${partnerApiKey}` |
| `ida.certificate.url` | Certificate fetch URL: `${mosip.base.url}/idauthentication/v1/internal/getCertificate` |
| `ida.authmanager.url` | Auth manager token URL: `${mosip.base.url}/v1/authmanager/authenticate/clientidsecretkey` |
| `ida.reference.id` | Reference ID for the IDA certificate (default: `PARTNER`). |
| `ida.captureRequest.uri` | Local SBI biometric capture URL (default: `http://127.0.0.1:4501/capture`). |

### Biometric Capture Properties

Each biometric modality (Finger, Iris, Face) has the following set of properties (prefix varies: `captureFinger`, `captureIris`, `captureFace`):

| Property Suffix | Description |
|-----------------|-------------|
| `.deviceId` | Physical device ID to request capture from. |
| `.deviceSubId` | Sub-ID of the device (e.g., `1` for slab fingerprint scanner). |
| `.timeout` | Capture timeout in milliseconds. |
| `.requestedScore` | Minimum quality score requested. |
| `.type` | Biometric type string: `"Finger"`, `"Iris"`, or `"Face"`. |
| `.bioSubType` | Default biometric sub-type (e.g., `"UNKNOWN"`). |
| `.domainUri` | Domain URI sent to SBI device for binding. |
| `.env` | Environment string (e.g., `"Staging"`). |
| `.name` / `.value` | Custom option name/value pairs included in the capture request. |

---

## 11. End-to-End Authentication Flow

The following describes the complete flow for a **biometric + OTP authentication** request:

```
User fills UIN/VID ──► User selects auth types (e.g., Finger + OTP)
         │
         ▼
1. [OTP Flow] onRequestOtp()
   ├─ Build OtpRequestDTO
   ├─ Sign request JSON (JWS RS256 with partner key)
   └─ POST to ida.otp.url
         │ User receives OTP on registered email/phone
         ▼
2. [Capture Flow] onCapture()
   ├─ captureFingerprint() / captureIris() / captureFace()
   │   └─ POST "CAPTURE" to local SBI at 127.0.0.1:4501/capture
   │       └─ Returns JWS-signed biometric data
   └─ combineCaptures() → stores JSON in `capture`
         │
         ▼
3. [Auth Request Flow] onSendAuthRequest()
   ├─ Build request block: {otp, timestamp, biometrics[...]}
   ├─ kernelEncrypt(request):
   │   ├─ genSecKey() → AES-256 session key
   │   ├─ symmetricEncrypt(request JSON) → encryptedIdentity
   │   ├─ getCertificate() ← fetch IDA public cert from MOSIP server
   │   ├─ asymmetricEncrypt(session key, IDA public key) → encryptedSessionKey
   │   ├─ symmetricEncrypt(SHA-256(request JSON)) → requestHMAC
   │   └─ SHA-256(IDA cert) → thumbprint
   ├─ Build AuthRequestDTO with encrypted fields
   ├─ sign(authRequestJSON) → JWS detached signature (RS256, partner cert in x5c)
   └─ POST to ida.auth.url with signature header
         │
         ▼
4. [Response Handling]
   ├─ Parse authStatus / kycStatus from response
   ├─ [eKYC only] decryptEkycResponse():
   │   ├─ Load EKYC partner private key
   │   ├─ decryptSecretKey(RSA-OAEP) → AES session key
   │   └─ AES-GCM decrypt KYC data → pretty-print JSON
   └─ Display result in responsetextField (green=success, red=failure)
```

### Security Notes

- All outbound requests are signed with the partner's RS256 private key; the IDA server verifies the signature using the certificate in the `x5c` JWS header.
- The identity block is protected by hybrid encryption: AES-256-GCM for the payload (confidentiality + integrity), RSA-OAEP for the AES key (key encapsulation).
- The HMAC (SHA-256 of the request, encrypted with the same AES key) provides an additional integrity check that the server can verify after decrypting the session key.
- SSL certificate validation is intentionally disabled for demo purposes (`turnOffSslChecking()`). This should **never** be done in production.
- The PKCS12 default password (`qwerty@123`) should be replaced via the `p12.password` property in any non-trivial deployment.
