# Agent Card Registry – 

A small demo of the Agent Card Registry pattern I’m proposing for A2A-style agents.

The goal is to show how **AgentCards become trustworthy** when they are:
1. **Ingested and governed by a registry**, and  
2. **Cryptographically bound to workload identities**,  
instead of being blindly trusted just because some workload happens to host a JSON file.

---

## 1. Problem Statement

In the current A2A mental model:

- An **AgentCard** is hosted by the agent implementation itself (e.g. `https://agent.example.com/.well-known/agent-card.json`).
- A **registry or client** fetches that card and uses it as the description of the agent.
- There is **no standardized mechanism** to ensure:
  - that the card came from a trusted publisher, or
  - that the runtime workload serving that card is actually the **legitimate agent**.

This creates two trust gaps:

1. **Ingestion gap (who can publish a card?)**  
   Any compromised or malicious workload can publish an AgentCard and claim to be “Agent X”. If a registry accepts that at face value, you’ve already lost.

2. **Runtime binding gap (who can *present* a card?)**  
   Even if a card is valid-looking, there is no guarantee that the workload you are connected to is the one that “owns” that card. A rogue service can host or replay a copied card and impersonate a legitimate agent.

**Net effect:**  
Relying on “workload-served AgentCards” plus URLs is not enough to prevent **spoofing, impersonation, or misconfiguration** in multi-tenant or adversarial environments.

---

## 2. Proposed Solution (High-Level)

Introduce an **Agent Card Registry** as a **central trust anchor**, and push both ingestion and runtime decisions through it.

Core idea:

> **Registry = source of truth for AgentCards**  
> **Workload identity = proof of who is running**  
> **Registry policy = decides whether that workload may present that card**

The demo will show:

1. **Secure ingestion**  
   - Only authenticated publisher identities (CI pipeline, operator, or attested workload) can register or update an AgentCard.
   - The registry validates the card (basic schema / optional signature) and stores an **authoritative copy**.
   - The registry records an **allowed workload identity** (or a set of them) for each card.

2. **Runtime binding check**  
   - When a client wants to call an agent, it:
     - Retrieves the AgentCard from the registry (not from the workload).
     - Connects to the agent endpoint and observes the **workload identity** (e.g. SPIFFE ID / mTLS subject / demo token).
     - Asks the registry:  
       _“Is workload identity **W** authorized to serve AgentCard **C**?”_
   - The registry returns **allow/deny** based on the stored binding.
   - Only if the binding matches does the client proceed with invocation.

This pattern cleanly separates:

- **Card provenance & ingestion trust**, and  
- **Runtime identity & binding trust**.

---

## 3. Demo Architecture

### Components

1. **Agent Card Registry Service**
   - REST API exposing:
     - `POST /agent-cards` – register/update an AgentCard (secure ingestion)
     - `GET /agent-cards/{id}` – fetch the canonical AgentCard
     - `POST /bindings` – bind AgentCard → workload identity (or done inline on registration)
     - `POST /check-binding` – given `{card_id, workload_identity}`, respond **allow/deny**
   - Stores data in a simple DB (SQLite for the demo):
     - `agent_cards` table (id, card JSON, publisher, created_at, etc.)
     - `bindings` table (card_id, workload_identity, status, etc.)

2. **Publisher / Operator**
   - A trusted identity that can call `POST /agent-cards`.
   - In the demo this can just be a static API key or “admin token”.
   - Represents either CI/CD or an operator onboarding an agent.

3. **Agent Workload (Remote Agent)**
   - A minimal HTTP server exposing a fake “A2A-like” endpoint.
   - Presents a **workload identity**:
     - In the real world: SPIFFE/mTLS/DID.
     - In the demo: a simple `X-Workload-ID` header or hard-coded token.
   - Does **not** serve its own card – the registry does.

4. **Client**
   - Discovers AgentCards via the registry:
     - `GET /agent-cards/{id}`
   - Uses the `url` contained in the card to contact the agent.
   - On connection, obtains the agent’s workload identity (simulated).
   - Calls the registry:
     - `POST /check-binding { card_id, workload_identity }`
   - Only invokes the agent if the registry returns **allow**.

---

## 4. Trust Model (What This Demo Proves)

### 4.1 Ingestion Trust

- The registry accepts an AgentCard only if the caller is authenticated as a **publisher**.
- The registry becomes the single **source of truth** for AgentCards:
  - Clients and other agents never trust arbitrary workload-hosted cards.
  - There is a clear audit trail: who registered which card, and when.

This addresses:  
> “How did the card get into the registry, and why should I trust it at all?”

### 4.2 Runtime Binding Trust

- Each AgentCard is **bound to one or more workload identities**.
- When a client connects to an agent and sees workload identity `W`, it:
  - asks the registry if `W` is authorized to present card `C`.
- If a rogue workload tries to impersonate the agent:
  - its identity `W'` will not match the registry binding for `C`,
  - and the registry will respond **deny**.

This addresses:  
> “Is the workload I’m currently talking to actually allowed to act as this agent?”

---

## 5. Demo Flow (End-to-End)

### Step 1 – Register an AgentCard

1. Publisher calls:
   ```http
   POST /agent-cards
   Authorization: Bearer PUBLISHER_TOKEN
   Content-Type: application/json

   {
     "id": "hello-agent",
     "url": "http://localhost:9000",   // agent endpoint
     "provider": "demo-org",
     "skills": ["echo", "sum"],
     "workload_identities": ["agent-hello-workload"]
   }
   ```
2. Registry:

- Validates the card.

- Stores it in agent_cards.

- Stores the binding ("hello-agent" ↔ "agent-hello-workload") in bindings.

### Step 2 – Client discovers the AgentCard

1. Client calls:
    ```
    GET /agent-cards/hello-agent
    ```
2. Registry returns the canonical AgentCard JSON.

### Step 3 – Client connects to the agent

Client reads url from the card: `http://localhost:9000`.

Client sends a request and receives a simulated workload identity:

e.g. the agent includes `X-Workload-ID: agent-hello-workload` in its response headers.

### Step 4 – Client verifies binding with the registry

1. Client calls:
    POST /check-binding
    Content-Type: application/json

    {
    "card_id": "hello-agent",
    "workload_identity": "agent-hello-workload"
    }
2. Registry looks up bindings and returns:
```
{ "allowed": true }
```

### Step 5 – Client invokes the agent

Since the binding check passed, the client now trusts:

the AgentCard (because it came from the registry), and

the workload identity (because registry confirmed the binding).

Client proceeds to use the agent’s skills.

If instead the workload identity had been evil-workload, the check-binding call would return allowed: false, and the client would refuse to invoke the agent.

## Scope of This Demo

This demo is intentionally minimal. It does not attempt to fully implement:

- real SPIFFE/mTLS or DIDs,

- full A2A protocol,

- complex RBAC/tenancy,

- artifact / supply-chain attestation.

It simply demonstrates the core pattern:

A Registry with secure ingestion + card↔workload binding is a much safer trust anchor than workload-served AgentCards.
Once this pattern is clear, you can plug in:

- real identity (SPIFFE, mTLS),

- real CI/CD publishers,

- A2A-compliant AgentCards and transports,


```mermaid
flowchart LR
    subgraph R["Quay (OCI Registry)"]
        Q[(AgentCard OCI Artifacts)]
    end

    subgraph ACR["Agent Card Registry (Trust Anchor & Policy)"]
        RC[Ingestion & Metadata Store]
        RB[Card ↔ Workload Identity Bindings]
        API[/REST API<br/>/agent-cards, /check-binding/]
    end

    subgraph AG["Agent Workload (Remote A2A Agent)"]
        AW[(A2A HTTP Server)]
        ID{{Workload Identity<br/>(SPIFFE/mTLS/Demo-ID)}}
    end

    P["Publisher / CI Pipeline"]
    C["Client / Caller (App or Agent)"]

    %% Ingestion path
    P -- "1. Push AgentCard as OCI artifact\n(signed JSON)" --> Q
    P -- "2. Register card_id + oci_digest\n+ allowed workload_identities" --> RC
    RC -- "3. Optional: Pull & verify\ncard from Quay" --> Q
    RC --> RB
    RC --> API
    RB --> API

    %% Discovery path
    C -- "4. Discover card\nGET /agent-cards/{card_id}" --> API
    API -- "5. Canonical AgentCard JSON\n(including agent URL)" --> C

    %% Runtime call path
    C -- "6. Invoke agent URL\n(A2A / HTTP)" --> AW
    AW -- "7. Present Workload Identity\n(SPIFFE/mTLS or demo header)" --> C

    %% Binding check
    C -- "8. POST /check-binding\n{card_id, workload_identity}" --> API
    API -- "9. allow / deny\nbased on stored binding" --> C

    %% Final invocation (only if allowed)
    C -- "10. Invoke skills\n(allowed only if binding ok)" --> AW