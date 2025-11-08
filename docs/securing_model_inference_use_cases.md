
# Securing Model Inference Use Cases with F5 Distributed Cloud API Security

This document outlines the configuration and testing procedures for securing Generative AI model inference endpoints deployed in complex environments like **Red Hat OpenShift AI (on ROSA)**, leveraging **F5 Distributed Cloud (XC) Web Application and API Protection (WAAP)** capabilities. The objective of every use case detailed below is to secure the inference endpoint from unauthorized access, prompt manipulation, and resource exhaustion.

---

## Step 0: Initial Load Balancer Configuration and Inference Endpoint Verification

Before applying specific security controls, the model inference service must be advertised via an **F5 Distributed Cloud HTTP Load Balancer (LB)** using **Origin Pools**.

### 1. Ensure F5XC Site Operational
Verify that the F5 Distributed Cloud site (or vK8s cluster) hosting the model service is operational.

### 2. Set up the HTTP Load Balancer
- Navigate to the **Multi-Cloud App Connect** service, then select **HTTP Load Balancers**.
- Click **Add HTTP Load Balancer** and provide a name and domain (e.g., `vllm-quantized.volt.thebizdevops.net`).

### 3. Configure the Origin Pool
- Click **Add Item** to create a new Origin Pool.
- Add an **Origin Server**: Select **K8s Service Name of Origin Server on given Sites**.
- Specify the service name corresponding to `llamastack.f5-ai-security` (using the format `servicename.namespace` if applicable, or select the relevant k8s service).
- Select the **Virtual Site type** as `shared/ves-io-all-res` and specify the appropriate **Port** (e.g., `8080` or the specific port the LLM stack listens on).
- **Save and Exit** the Origin Pool and HTTP Load Balancer.

### Verification of Inference Endpoint Access

Once the LB is saved, verify that the inference endpoint is correctly exposed:

```bash
curl -sS http://vllm-quantized.volt.thebizdevops.net//v1/openai/v1/models | jq
```

**Expected Output:**

```json
{
  "data": [
    {
      "id": "remote-llm/RedHatAI/Llama-3.2-1B-Instruct-quantized.w8a8",
      "object": "model",
      "created": 1762644418,
      "owned_by": "llama_stack"
    },
    {
      "id": "sentence-transformers/all-MiniLM-L6-v2",
      "object": "model",
      "created": 1762644418,
      "owned_by": "llama_stack"
    }
  ]
}
```

---

## Use Case 1: Protecting the Inference Endpoint Against Prompt Manipulation and Injection Attacks

Model inference endpoints that accept user prompts are susceptible to injection attacks (like **Cross-Site Scripting or SQLi**), often referred to as **prompt injection or manipulation**. The **Web Application Firewall (WAF)** inspects requests and responses using signatures and behavioral-based threat detection to block these risks.

### Detailed Configuration Steps (WAF)

1. **Access LB Configuration**: Navigate to **Web App & API Protection > HTTP Load Balancers**. Select the LB → **Manage Configuration** → **Edit Configuration**.  
2. **Enable App Firewall**: In the **Web Application Firewall** section, enable it and click **Add Item** to configure a new WAF object.  
3. **Set Enforcement Mode**: Set to **Blocking**.  
4. **Customize Detection Settings**: Select **Custom** to override defaults.  
5. **Configure Attack Types**: Disable specific attack types (optional).  
6. **Set Signature Accuracy**: Enable **High**, **Medium**, and **Low**.  
7. **Define Blocking Response**: Choose **Custom** → `403 Forbidden`.  
8. **Save Configuration**.  

### Traffic Generation and Confirmation

- Run a test request with injection payloads:  
  ```bash
  curl -X POST http://vllm-quantized.volt.thebizdevops.net/v1/chat/completions     -H "Content-Type: application/json"     -d '{"prompt": "' OR 1=1 --"}'
  ```

- Expected response: `403 Forbidden`
- Check **Security Dashboard > Security Events** for WAF logs.
- To block repeat offenders, use **Add to Blocked Clients** under **Malicious User Detection (MUD)**.

---

## Use Case 2: Enforcing API Specification and Preventing Shadow APIs

**API Protection** secures inference endpoints using a defined **OpenAPI (Swagger)** specification. **API Discovery** uses ML to detect shadow APIs and sensitive data exposure.

### Detailed Configuration Steps (API Protection)

1. **Upload Swagger File** → `Swagger Files > Add Swagger File`.  
2. **Create API Definition** → `API Definition > Add` → attach Swagger spec.  
3. **Attach Definition to LB** → `Edit Configuration > API Protection`.  
4. **Create Service Policy** → under **Common Security Controls > Apply Specified Service Policies**:  
   - **Rule 1 (Deny Non-API)**:  
     - Action: **Deny**  
     - Path: `/v1/`  
     - **Invert String Matcher** = true (blocks undocumented paths).  
   - **Rule 2 (Allow Other)**: Action = **Allow**  
5. **Save Configuration**.

### Traffic Generation and Confirmation

- Generate legitimate API traffic (e.g., via Swagger UI or Postman).
- Attempt to access undocumented path → access should be forbidden.
- Review **API Discovery Dashboard** for new and shadow APIs.

---

## Use Case 3: Mitigating Automated Attack Traffic and Excessive Requests

Inference endpoints must be protected from **resource exhaustion** and **bot traffic**.

### Detailed Configuration Steps (Bot Protection)

1. **Enable Bot Defense** → in **HTTP LB > Edit Configuration** → **Bot Protection**.  
2. Add **App Endpoint**: Path `/v1/chat/completions`, methods `PUT`, `POST`.  
3. **Mitigation Action**: **Block (403)**.  
4. **Save Configuration**.

### Detailed Configuration Steps (Rate Limiting and DDoS)

1. **Enable IP Reputation** → choose threat categories (Spam, DoS, Tor, Botnets).  
2. **Configure Rate Limiting** → 10 requests/sec, burst multiplier 5.  
3. **Add DDoS Rule** → block IP `203.0.113.0/24`.  
4. **Save Configuration**.

### Traffic Generation and Confirmation

- Simulate bots via Python scripts or load tests (`ab`, `wrk`, etc.).  
- Observe **403 Forbidden** for blocked requests.  
- Review **Bot Defense** and **DDoS** analytics dashboards for events.

---

## Analogy

Securing a model inference endpoint with F5 Distributed Cloud is like protecting a **high-security vault** housing your **LLM**:

- **WAF**: Acts as biometric scanner and metal detector.  
- **API Protection**: Ensures only documented doors (API paths) are accessible.  
- **Bot Defense / Rate Limiting**: Crowd control, preventing overload by malicious actors.

Together, they create a layered, intelligent security system ensuring **only legitimate, well-formed requests** reach your AI model.
